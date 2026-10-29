# bluesky-trends-deploy

Kubernetes manifests for [bluesky-trends](https://github.com/dedyoc/bluesky-trends), a
real-time analytics pipeline on the Bluesky firehose, running as tenant `bsky` on a
single-node K3s homelab.

- **Code** (ingest, schemas, ClickHouse SQL, Dagster, dbt, Flink job) lives in
  `dedyoc/bluesky-trends`. Images are built there.
- **This repo** holds only namespaced manifests. ArgoCD syncs `main` into namespace `bsky`.
- **The platform** (namespace, memory quota, ArgoCD project, Kafka/ClickHouse/Flink operators)
  is managed in a separate platform repo. This repo can't create cluster-scoped objects.

Stack: Strimzi Kafka (KRaft), Karapace, ClickHouse, Grafana, MinIO + Iceberg, Dagster + dbt,
Flink, Prometheus. Progress and design decisions: [`docs/STATE.md`](docs/STATE.md),
[`docs/decisions.md`](docs/decisions.md).

No secrets live here: they're created out of band and referenced by name.
