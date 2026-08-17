# Thanos Lab — Object Storage Tasks (20–25)

> Covers the object-storage half of the lab: the Compactor, then replacing
> MinIO with a real AWS S3 bucket. Tasks 1–19 are distributed separately.

---

## Task 20 – Deploy the Compactor and Test Durability

**Goal:** Manage historical metrics and prove that metrics survive Prometheus restarts.

**Steps**

Deploy the Thanos Compactor:

```bash
kubectl apply -f objstore\53-compactor.yaml
```

Wait for it:

```bash
kubectl rollout status deployment/thanos-compact -n monitoring
```

Check its logs:

```bash
kubectl logs -n monitoring deploy/thanos-compact --tail=30
```

Now delete both Prometheus Pods:

```bash
kubectl delete pod -n monitoring -l replica=a
```

```bash
kubectl delete pod -n monitoring -l replica=b
```

Wait for the Prometheus deployments to recover. Restart the required port-forwards.

In Thanos Query, select a time range from before the Prometheus restart.

**Verify**

Historical metrics should still exist. The data is now coming from:

```
MinIO → Store Gateway → Thanos Query
```

This proves that the metrics no longer depend only on the local Prometheus disk.

> **Note:** `objstore/50-minio.yaml` is no longer in this repo — the lab was
> switched to S3-only. If you want to run the MinIO version of this task,
> restore that file first. Otherwise read "MinIO" here as "your object store"
> and go straight to Task 21.

---

## Thanos with Real AWS S3

The next tasks replace MinIO with a real AWS S3 bucket. The architecture is almost identical:

```
Prometheus → Thanos Sidecar → AWS S3 → Store Gateway → Thanos Query
```

Before starting, choose:

- `BUCKET` — your globally unique S3 bucket name
- `REGION` — your AWS region

Example: `BUCKET=thanos-lab-daniel-48291`, `REGION=eu-west-1`

---

## Task 21 – Create the S3 Bucket and IAM User

**Goal:** Create AWS storage and permissions for Thanos.

**Steps**

Create the S3 bucket. For `us-east-1`, use:

```bash
aws s3api create-bucket --bucket BUCKET --region us-east-1
```

> For any other region you must add
> `--create-bucket-configuration LocationConstraint=REGION`.
> In `us-east-1` that flag causes `InvalidLocationConstraint`.

Verify the bucket:

```bash
aws s3 ls
```

Create an IAM user:

```bash
aws iam create-user --user-name thanos-lab
```

Create `thanos-policy.json`:

```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": ["s3:ListBucket"],
      "Resource": "arn:aws:s3:::BUCKET"
    },
    {
      "Effect": "Allow",
      "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
      "Resource": "arn:aws:s3:::BUCKET/*"
    }
  ]
}
```

Attach the policy:

```bash
aws iam put-user-policy --user-name thanos-lab --policy-name thanos-bucket --policy-document file://thanos-policy.json
```

Create an access key:

```bash
aws iam create-access-key --user-name thanos-lab
```

**Verify**

Save the `AccessKeyId` and `SecretAccessKey`. The Secret Access Key is only displayed once.

The IAM user only requires: `s3:ListBucket`, `s3:GetObject`, `s3:PutObject`, `s3:DeleteObject`.

> `put-user-policy` **replaces** the inline policy of that name. If you run
> this twice with different bucket names, only the last one is granted — and
> the Store Gateway will crash with `not authorized to perform: s3:ListBucket`.

---

## Task 22 – Add AWS S3 Credentials to Kubernetes

**Goal:** Configure Thanos to connect to the real S3 bucket.

**Steps**

Create `objstore-s3.yml` (a template lives at `k8s/objstore/objstore-s3.yml.example`):

```yaml
type: S3
config:
  bucket: BUCKET
  endpoint: s3.REGION.amazonaws.com
  region: REGION
  access_key: AKIA................
  secret_key: ................................
```

Delete the previous object-storage Secret:

```bash
kubectl delete secret thanos-objstore -n monitoring --ignore-not-found
```

Create the new Secret:

```bash
kubectl create secret generic thanos-objstore -n monitoring --from-file=objstore.yml=objstore-s3.yml
```

Check it:

```bash
kubectl get secret thanos-objstore -n monitoring
```

**Verify**

The Secret should exist in the `monitoring` namespace.

Notice that both MinIO and AWS use `type: S3`. The main differences are the endpoint and credentials.

In production on EKS, IAM Roles for Service Accounts or Pod Identity would normally be preferred over long-lived access keys.

> **Never commit `objstore-s3.yml`.** It contains a live AWS key, and bots
> scrape public GitHub for `AKIA...` within minutes of a push.

---

## Task 23 – Upload Prometheus Blocks to AWS S3

**Goal:** Send Prometheus historical metrics to the real S3 bucket.

**Steps**

Restart both Prometheus deployments so the Sidecars reload the Secret:

```bash
kubectl rollout restart deployment/prometheus-a -n monitoring
```

```bash
kubectl rollout restart deployment/prometheus-b -n monitoring
```

Wait for them:

```bash
kubectl rollout status deployment/prometheus-a -n monitoring
```

```bash
kubectl rollout status deployment/prometheus-b -n monitoring
```

Check the Sidecar logs:

```bash
kubectl logs -n monitoring deploy/prometheus-a -c thanos-sidecar --tail=20
```

Check whether Prometheus has created a block:

```bash
kubectl exec -n monitoring deploy/prometheus-a -c prometheus -- ls /prometheus
```

> **Be patient.** The restart wipes the pod's `emptyDir`, so Prometheus
> collects from zero and needs **8–10 minutes** before it cuts its first
> 5-minute block. Until then you will see only `chunks_head`, `lock` and
> `wal` — that is normal, not a failure. A ULID directory means a block
> exists and the sidecar has something to upload.

When a ULID directory appears, check:

```bash
kubectl logs -n monitoring deploy/prometheus-a -c thanos-sidecar --tail=30
```

**Verify**

Look for `upload new block`. Now check AWS:

```bash
aws s3 ls s3://BUCKET/
```

You should see ULID-named prefixes. A Thanos block contains `chunks/`, `index` and `meta.json`.

---

## Task 24 – Read Historical Metrics from AWS S3

**Goal:** Configure Store Gateway and Compactor to use AWS S3.

**Steps**

Deploy them:

```bash
kubectl apply -f objstore\52-store-gateway.yaml
```

```bash
kubectl apply -f objstore\53-compactor.yaml
```

Restart both components so they re-read the Secret:

```bash
kubectl rollout restart deployment/thanos-store deployment/thanos-compact -n monitoring
```

> A Thanos component reads its objstore config **only at startup**. Changing
> the Secret does nothing until the pod restarts.

Check the Thanos Query configuration:

```bash
kubectl get deployment thanos-query -n monitoring -o jsonpath="{.spec.template.spec.containers[0].args}"
```

Make sure `--endpoint=thanos-store.monitoring.svc.cluster.local:10901` exists. If it is missing, add it:

```bash
kubectl patch deployment thanos-query -n monitoring --type=json --patch-file objstore\54-add-store-endpoint.json
```

> The inline form `-p "[{\"op\":\"add\",...}]"` **fails on PowerShell** with
> `unable to parse "[{\\": yaml: line 1: did not find expected ',' or '}'`,
> and it fails quietly — Thanos Query keeps working, just with no historical
> data. Use the patch file above, then re-run the `jsonpath` command to
> confirm the endpoint really landed.
>
> `kubectl apply -k .` overwrites this patch. If the store disappears from
> the Endpoints tab, apply it again.

Check Store Gateway logs:

```bash
kubectl logs -n monitoring deploy/thanos-store --tail=20
```

**Verify**

Look for `successfully synchronized block metadata` with `returned=` greater than 0.

You can also inspect the S3 blocks directly through Thanos:

```bash
kubectl exec -n monitoring deploy/thanos-store -- thanos tools bucket ls --objstore.config-file=/etc/thanos/objstore.yml
```

The command should list the blocks stored in S3 and finish with `ls done objects=N`.

---

## Task 25 – Prove the Metrics Survive

**Goal:** Prove that historical metrics survive even when both Prometheus Pods are destroyed.

**Steps**

First query Thanos for `up{job="podinfo"}` and make sure historical data exists.

Now delete both Prometheus Pods:

```bash
kubectl delete pod -n monitoring -l app=prometheus
```

Wait for both deployments:

```bash
kubectl rollout status deployment/prometheus-a -n monitoring
```

```bash
kubectl rollout status deployment/prometheus-b -n monitoring
```

Check the new Prometheus storage — it should be empty:

```bash
kubectl exec -n monitoring deploy/prometheus-a -c prometheus -- ls /prometheus
```

Restart your Thanos port-forward if necessary:

```bash
kubectl port-forward -n monitoring svc/thanos-query 10902:10902
```

Open http://localhost:10902, select **Last 1 hour** and query `up{job="podinfo"}`.

**Verify**

Metrics from before the Prometheus restart should still be available, served through:

```
AWS S3 → Store Gateway → Thanos Query
```

The metrics survived the Prometheus servers that originally collected them.

---

## Cleanup AWS Resources

```bash
aws s3 rm s3://BUCKET/ --recursive
```

```bash
aws s3api delete-bucket --bucket BUCKET --region REGION
```

```bash
aws iam delete-user-policy --user-name thanos-lab --policy-name thanos-bucket
```

```bash
aws iam list-access-keys --user-name thanos-lab
```

```bash
aws iam delete-access-key --user-name thanos-lab --access-key-id AKIA......
```

```bash
aws iam delete-user --user-name thanos-lab
```

> Storage costs pennies. The **access key** is the real risk — delete it.

---

## Final Architecture

**MinIO Lab**

```
Prometheus A ── Sidecar A ──┐
                            ├──→ MinIO ──→ Store Gateway ──→ Thanos Query ──→ Grafana
Prometheus B ── Sidecar B ──┘
```

**AWS Version**

```
Prometheus A ── Sidecar A ──┐
                            ├──→ AWS S3 ──→ Store Gateway ──→ Thanos Query ──→ Grafana
Prometheus B ── Sidecar B ──┘
```

The important lesson:

- With MinIO: `Thanos → S3 API → MinIO`
- With AWS: `Thanos → S3 API → AWS S3`

The Thanos architecture stays the same; only the object-storage configuration changes.
