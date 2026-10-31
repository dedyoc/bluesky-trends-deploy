# STATE.md — tenant `bsky` — last updated: 2026-10-29

> Current facts about the `bsky` tenant on node0. Platform facts (node, RAM budget, disk,
> network) live in `~/homelab/docs/STATE.md`; cite them, don't copy them. Decisions →
> `decisions.md`. Closed items are deleted, not ticked.

## Slice position

| Slice | Status | Proof |
|---|---|---|
| 0 Skeleton | ✅ 2026-10-29 | Application `bsky` `Synced`/`Healthy` at `ee8f01f`; quota and LimitRange live (homelab `field-notes.md`) |
| 1 Ingest | ▶ next | Blocked on the Strimzi operator (Waiting on platform) |
| 2 Hot path | — | |
| 3 Batch | — | |
| 4 Flink | — | |
| 5 Metrics | — | |
| 6 Drills | — | |

## Platform contract (homelab ADR-042, ADR-043)

- Namespace `bsky`; `AppProject` `bsky`: source this repo only, destination `bsky` only, no
  cluster-scoped kinds, `ResourceQuota`/`LimitRange` blacklisted.
- `ResourceQuota` `bsky-memory`: `requests.memory` 10Gi, `limits.memory` 14Gi, 12 PVCs.
- `LimitRange` `bsky-defaults`: per-container default 128Mi request / 512Mi limit.
- Application `bsky`: `main`, path `.`, automated, prune + selfHeal, no finalizer.
- **No backups** (homelab ADR-044): restic excludes `*_bsky_*`. Every volume here is
  losable; a rebuild brings the tenant back empty, raw Iceberg archive included.
- **Jetstream is reachable from node0's host** (2026-10-29): websocket `101`, ~27 KB/s of posts
  only. From a pod: not tested; expect the same (pods egress via the node). Use `curl --http1.1`
  to probe; plain curl gives a misleading `426`.
- First sync of a new tenant can show `InvalidSpecError … project bsky which does not exist`
  for a few minutes. It clears by itself; wait one reconcile.

## Deployed

Nothing yet.

## RAM table (Mi)

**Estimates only, from typical footprints, not measured on node0.** Replace each row with
`kubectl top pod` (steady state, after ~24h) when its slice lands, and set requests/limits from
the measurement. Operators run outside `bsky` and are outside the quota (homelab accounts them).

| Component | Slice | Est. resident | Request / limit (plan) | Notes |
|---|---|---|---|---|
| Kafka, 1 node, KRaft combined | 1 | ~1200 | 1280 / 2048 | heap 768m–1g |
| Strimzi entity operator (topic op only) | 1 | ~300 | 256 / 512 | created by Strimzi *in `bsky`*: counts against the quota. Disable the user operator |
| Karapace | 1 | ~200 | 192 / 384 | Python |
| Postgres (cursor; later Dagster + Iceberg catalog) | 1 | ~150 | 128 / 512 | one instance, several DBs |
| ingest | 1 | ~100 | 96 / 256 | |
| ClickHouse | 2 | ~1500 | 1536 / 3072 | cap `max_server_memory_usage` below the limit |
| Grafana | 2 | ~150 | 128 / 256 | |
| MinIO | 3 | ~300 | 256 / 512 | |
| Iceberg REST catalog | 3 | ~350 | 320 / 640 | JVM |
| Dagster webserver + daemon + user code | 3 | ~900 | 768 / 1536 | each run pod adds ~0.5–1 Gi while it runs |
| Flink JobManager + 1 TaskManager (PyFlink) | 4 | ~2500 | 2560 / 3072 | TM = JVM + Python worker |
| Prometheus | 5 | ~400 | 384 / 768 | short retention |
| **Total (plan)** | | **~7950** | **~7900 / ~13550** | fits 10Gi / 14Gi with little room for run pods |

## Secrets inventory

Names, keys and how to regenerate. **Never values.** Created by hand on node0; a rebuild
re-runs these steps.

| Secret | Keys | Used by | Regenerate |
|---|---|---|---|
| *(none yet)* | | | |

## Waiting on platform (take to a homelab session)

- **Strimzi operator** for slice 1: version, install method, `watchNamespaces: [bsky]`.

## Open items

- [ ] **Images:** no CI in `bluesky-trends` yet. Slice 1 needs `ghcr.io/dedyoc/…` ingest image,
  public, pinned by digest here. That work happens in the `bluesky-trends` session.
