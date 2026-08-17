# Thanos demo — one global view over two Prometheus instances

A small, showable demo of what Thanos actually does: take several separate
Prometheus servers and query them as if they were one.

```bash
docker compose up -d
```

| What | Where |
|---|---|
| **Thanos Query** | http://localhost:10902 ← the demo lives here |
| prometheus-a | http://localhost:9090 |
| prometheus-b | http://localhost:9091 |
| Grafana | http://localhost:3001 (no login) |
| podinfo (the target) | http://localhost:9898/metrics |

Port 3000 is deliberately left free so this can run alongside the tracing lab.

## The setup

```
podinfo   <-- scraped by BOTH prometheus instances

prometheus-a ── sidecar-a ─┐
                           ├──> thanos-query ──> Grafana
prometheus-b ── sidecar-b ─┘
```

`prometheus-a.yml` and `prometheus-b.yml` are **identical except one label**:

```yaml
external_labels:
  cluster: demo
  replica: a        # and b in the other file
```

Both scrape the same target, so you have two independent copies of the same
data. That is the situation Thanos exists to solve.

## The demo — three moves

**1. Two Prometheus servers that know nothing about each other**

Open http://localhost:9090 and http://localhost:9091, run `up` in each.
Same metrics, two separate servers, two separate web UIs. Neither can answer
a question about the other.

**2. Show that Thanos sees both**

Open **http://localhost:10902** → the **Endpoints** tab. Both sidecars are
listed UP, with the labels that distinguish them:

```
sidecar-a:10901   UP   cluster="demo"  replica="a"
sidecar-b:10901   UP   cluster="demo"  replica="b"
```

**3. One query across both**

Go to the **Graph** tab and run:

```
up{job="podinfo"}
```

You get **ONE** series - because the checkbox **Use Deduplication** is
ticked BY DEFAULT. Untick it and re-run: now you get **two**, `replica="a"`
and `replica="b"`.

Do it in that order - start deduplicated, then untick to reveal the two
copies underneath. Thanos knows they are duplicates because of
`--query.replica-label=replica`.

**4. Kill one and watch it keep working**

```bash
docker compose stop prometheus-a
```

http://localhost:9090 is dead. Run the query in Thanos Query again — it
still returns data, served entirely by replica B. That is the HA argument
for running two Prometheis in the first place.

```bash
docker compose start prometheus-a
```

## Grafana

http://localhost:3001 has a datasource called **Thanos** pointing at
`thanos-query:10902`. Note what it is configured as: **type `prometheus`**.
Thanos Query speaks the Prometheus HTTP API, so Grafana has no idea it is
talking to anything other than a Prometheus. Nothing in your dashboards
needs to change to adopt Thanos.

## What this demo does NOT show

Thanos' other half is **long-term storage**: the sidecars upload 2h blocks
to object storage (S3/GCS/MinIO), a **Store Gateway** serves them back, and
a **Compactor** compacts and downsamples them. That needs a bucket, so it is
left out here to keep the demo to one command.

Without object storage you are seeing the *global view* and *HA dedup* half
of Thanos, which is the half that fits on a slide.

## Verified

Run end to end on 2026-08-12 (Docker 29.5.2):

- all 7 containers start
- `/api/v1/stores` shows both sidecars connected to Thanos Query
- `dedup=false` returns 2 series (`replica=a`, `replica=b`);
  `dedup=true` returns 1 with the replica label stripped
- with `prometheus-a` stopped, `localhost:9090` is dead but Thanos Query
  still answers from replica B

## Teardown

```bash
docker compose down -v
```
