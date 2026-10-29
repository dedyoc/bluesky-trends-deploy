# CLAUDE.md — bluesky-trends-deploy (tenant `bsky` on node0)

Loaded by Claude Code at the start of every session in this repo. Created 2026-10-28 from the
platform session in `~/homelab` (homelab ADR-042, ADR-043).

## Start of every session

1. Read `docs/STATE.md` (this tenant's current facts and slice position).
2. When a question touches node0 itself (RAM budget, disk, versions, network), read
   `~/homelab/docs/STATE.md`. It's canonical for the platform; don't restate its facts here.
3. If either looks stale or contradicts what you observe, say so before acting.

## What this repo is

Kubernetes manifests for the data tenant **`bsky`**: the `bluesky-trends` pipeline running on
node0. Three repos, three jobs:

| Repo | Holds | Who edits it |
|---|---|---|
| `~/bluesky-trends` (`dedyoc/bluesky-trends`) | code: ingest, schemas, ClickHouse SQL, Dagster, dbt, Flink job, Dockerfiles, CI | its own Claude session (its own `CLAUDE.md`, learning mode) |
| **this repo** (`dedyoc/bluesky-trends-deploy`, public) | namespaced YAML for `bsky` + tenant docs | this session |
| `~/homelab` (`dedyoc/homelab`) | the platform: Namespace, quota, `AppProject`, `Application`, operators | **read-only from here** |

ArgoCD `Application` `bsky` (defined in homelab `apps/tenants/bsky/`) syncs this repo's root
`kustomization.yaml` from `main` into namespace `bsky`, automated with **prune + selfHeal**.
`main` is production: a merge is live within ~3 min, and a deleted manifest is deleted from
the cluster, data included.

## Working mode: build first, explain after

The owner is a junior Data Engineer preparing for interviews. They want this running fast, then
want to be able to explain every piece. This repo **does not** use homelab's Socratic mode:

- You write the manifests. One slice at a time, each one a commit-sized batch.
- When you introduce a concept (KRaft, consumer groups, watermarks, operators, PVC binding),
  give a 2–3 line analogy or example first, then the formal version. Keep it short.
- After each **working** slice, add an entry to `docs/interview-notes.md` (format is in the
  file). Do this before starting the next slice; a slice isn't done without it.
- Still don't gold-plate: 1 replica, no HA, no in-cluster TLS, no NetworkPolicy, no
  autoscaling, unless a slice can't work without it. Say so when the owner drifts there.
- Never invent YAML fields, CR fields, Helm values or config keys. Verify against the installed
  CRD (`kubectl explain`), `helm show values <chart> --version <v>`, or the docs for the
  pinned version, and say which you used.

## Facts that shape every manifest

Canonical source: `~/homelab/docs/STATE.md`. Summary as of 2026-10-28:

- **One node.** node0: K3s v1.36.4+k3s1, Fedora Server 44, SELinux enforcing, cgroup v2.
- **RAM is the binding constraint.** The platform gives `bsky` a quota you can't change from
  here: `requests.memory` **10Gi**, `limits.memory` **14Gi**, **12 PVCs**. A LimitRange
  defaults containers to 128Mi request / 512Mi limit. **Every container you write sets memory
  requests and limits explicitly.** A pod the quota refuses never appears in `get pods`; it
  shows up as an event on its ReplicaSet or StatefulSet: `kubectl -n bsky get events`.
- **JVMs:** set the heap explicitly (`-Xmx`, or the operator's heap field) well below the
  container limit; off-heap and metaspace need the rest. Kafka, Strimzi's entity operator, the
  Iceberg REST catalog and Flink are JVMs. Karapace and Dagster are Python.
- **Storage:** `local-path` only. It doesn't enforce PVC sizes, can't expand and has no
  snapshots. Every PV lives on node0's 150G root filesystem, together with the OS and images.
  **Anything that grows gets a retention:** Kafka `retention.bytes`/`retention.ms` per topic,
  ClickHouse TTLs, a MinIO lifecycle or a bounded archive. When root fills to 95%, the kubelet
  evicts pods.
- **Backups: none, by decision** (homelab ADR-044). restic excludes every `bsky` volume.
  Treat all data here as losable: anything that must survive a dead disk lives in Git or gets
  re-collected. If a slice would hold something irreplaceable, stop and say so.
- **Network:** outbound only. No inbound ports, ever. Jetstream (`wss://…bsky.network`,
  outbound :443) is **not yet proven from node0**: the building network filters in ways we
  can't see. Image pulls from `ghcr.io` work.
- **UIs (Grafana, Dagster, MinIO console):** no Ingress. Memos owns `/` on every host (homelab
  ADR-037) and there is no domain yet (parked). Reach a UI from the laptop with an SSH tunnel
  plus a port-forward on node0, for example:
  `ssh.exe -L 3000:127.0.0.1:3000 node0 'kubectl --kubeconfig ~/.kube/config -n bsky port-forward svc/grafana 3000:3000'`.
  Avoid local ports 8998–9097: Windows reserves them.
- **Images:** public, on `ghcr.io/dedyoc/…`, built by CI in `bluesky-trends` (not set up yet),
  pinned **by digest** here. node0 has no registry and builds nothing.
- **Secrets:** this repo is public, so **no Secret goes in it, encrypted or not**. The owner
  creates Secrets by hand on node0 (`kubectl -n bsky create secret generic …`). Manifests
  reference them by name. `docs/STATE.md` → Secrets inventory records name, keys and how to
  regenerate each one, **never values**. If a Secret can't be regenerated, stop and say so:
  that fires homelab's SOPS trigger (ADR-029).

## Platform boundary

- **No cluster-scoped kinds** in this repo: no CRDs, Namespaces, ClusterRoles,
  ClusterRoleBindings, webhooks, StorageClasses, PriorityClasses. The `AppProject` rejects
  them, and **one rejected manifest fails the whole sync**, so the other components stop
  updating too.
- **No `ResourceQuota` or `LimitRange`.** They are platform-owned and blacklisted.
- **Operators come from homelab** (ADR-043), each watching `bsky` only. Before writing a CR,
  check its CRD is installed and read the schema at the installed version:
  `ssh.exe node0 'kubectl --kubeconfig ~/.kube/config explain kafka.spec.kafka --recursive' | less`.
- **Helm charts** (Dagster, maybe Grafana or MinIO) can't be pulled at sync time: the
  `AppProject` allows only this repo as a source. Render with `helm template` at a pinned chart
  version, commit the output under `<component>/vendor/` with the values file next to it, and
  note the exact command in the component's `kustomization.yaml` header. Re-render on upgrade
  and read the diff. Strip any cluster-scoped objects from the render (see above).
- **When a slice needs the platform** (an operator, a quota change, a restic exclude), don't
  edit `~/homelab`. Add it to `docs/STATE.md` → Waiting on platform, with what, why and the
  RAM cost, then stop. The owner takes it to a homelab session.

## Safety rules (non-negotiable)

1. **Never `git commit` or `git push` without the owner's explicit go-ahead in this session.**
2. **node0 is read-only for you.** Over `ssh.exe node0 '…'` you may run `kubectl get`,
   `describe`, `logs`, `top`, `explain` and `get events`, always with
   `--kubeconfig ~/.kube/config`. Anything that changes state (creating a Secret, deleting a
   pod for a drill, `exec` that writes, `port-forward` the owner will use) goes to the owner as
   a labelled block.
3. **Never touch secrets.** Don't read `~/.kube/config` or `~/.config/gh/`, and don't run
   `kubectl get secret -o yaml/json`, `describe secret`, or `exec … env` on a pod that mounts one.
4. **Never `kubectl apply/edit/delete/patch`.** Git is the only path to the cluster.
5. **Data-loss diffs get called out.** Before proposing any diff that removes or renames a
   StatefulSet, PVC, `Kafka`/`KafkaNodePool`, `ClickHouseInstallation` or MinIO volume, say in
   bold that prune will delete its data, and ask.

## Commands the owner runs on node0

They run as `dedyoc`. `kubectl` needs no `sudo`. Make them paste-safe: label read-only vs act;
split check blocks from act blocks; never put an interactive prompt mid-block. If `sudo` is
ever needed, have them run `sudo -v` alone first. **No heredoc act blocks:** write the script
to a file, then run it with `</dev/null` (a prompt once swallowed a heredoc). Secret creation
reads values from `openssl rand` or `read -s`, never from shell history.

## Roadmap: slices, in order

Each slice is done when it works on node0, its proof is written in `docs/STATE.md`, its
`kubectl top` numbers replace the estimates there, and `docs/interview-notes.md` has its entry.
No slice is pre-built "because we'll need it".

0. **Skeleton.** Root `kustomization.yaml` (empty) synced; `bsky` `Synced`/`Healthy`; quota
   visible (`kubectl -n bsky describe resourcequota`).
1. **Ingest** *(platform: Strimzi)*. Kafka in KRaft mode, one node with combined
   broker+controller roles (`KafkaNodePool`); `KafkaTopic`s `bsky.posts.v1`,
   `bsky.likes.v1`, `bsky.follows.v1`, `bsky.dlq.v1`, each with a retention; Karapace schema
   registry; Postgres for the Jetstream cursor; the ingest Deployment (image from
   `bluesky-trends` CI). **Proof:** events arrive; delete the ingest pod, it resumes from the
   cursor with no gap; a schema-invalid record lands in the DLQ.
2. **Hot path** *(platform: ClickHouse operator)*. `ClickHouseInstallation` (1 shard, 1
   replica, `max_server_memory_usage` below the limit); the v1 DDL (Kafka engine + MVs) from
   `bluesky-trends/clickhouse/init`; Grafana with the provisioned dashboards. **Proof:** live
   dashboard over the tunnel. *Resume bullets 1–2 are true from here.*
3. **Batch.** MinIO; Iceberg REST catalog **backed by Postgres (JDBC), not SQLite** (dev's
   SQLite catalog hit `SQLITE_BUSY` on concurrent commits); Dagster (vendored Helm render,
   Postgres for its storage); dbt inside the Dagster user-code image. **Proof:** bronze →
   landing → marts materialize, asset checks pass, one backfill from Iceberg.
4. **Flink** *(platform: Flink Kubernetes Operator, and cert-manager or its webhook off:
   decide there)*. A `FlinkDeployment` for the v3 PyFlink trend job once `bluesky-trends`
   finishes it: event-time windows, watermarks, checkpoints to MinIO, sink to ClickHouse.
   **Proof:** kill the TaskManager, the job restores from its checkpoint.
5. **Metrics.** A namespaced Prometheus, small retention, scraping Strimzi/Kafka exporter
   (consumer lag), ClickHouse, Flink and Dagster; Grafana panels. **Proof:** a lag panel that
   moves when you scale ingest to 0.
6. **Drills**: broker restart, a Kafka PVC filling up, node reboot with everything running.
   These make the best interview stories; record them in `docs/interview-notes.md`.

## Porting from docker-compose: known differences

- Dev runs **Redpanda**, which has a built-in schema registry. Here it's Strimzi Kafka, plus
  **Karapace** as its own Deployment. `redpanda-init`'s topic creation becomes `KafkaTopic` CRs.
- `bluesky-trends` docs assume namespaces `kafka`/`minio` and a repo named `homelab-ops`.
  Both are wrong: everything runs in `bsky`, and this repo is `bluesky-trends-deploy`. Note it
  for the `bluesky-trends` session; don't edit that repo from here.
- ClickHouse init scripts only run on a fresh volume (seen in dev). On K8s, decide in slice 2
  how DDL is applied and re-applied (a Job, or the operator's mechanism), and make it idempotent.
- Service DNS inside the namespace is `<svc>.bsky.svc.cluster.local`; config that hardcodes
  `localhost:<port>` from compose needs an env override in the ingest/Dagster images.

## Docs in `docs/`: one home per kind of fact

- `docs/STATE.md`: **current facts only**: slice status, what's deployed (with image digests),
  RAM table, Secrets inventory, Waiting on platform, open items. Closed items are deleted.
  Bump `last updated:`.
- `docs/decisions.md`: this tenant's ADR log (`ADR-001`, …), append-only, with an Index table.
  Platform decisions are cited as "homelab ADR-0NN" and never copied here. Never invent a
  rationale; if the owner doesn't remember, write "reason not recorded".
- `docs/interview-notes.md`: per-slice talking points.

After editing docs: `grep -cE '^### ADR-0' docs/decisions.md` must equal
`grep -cE '^\| 0[0-9]{2} \|' docs/decisions.md`.

## Response format

- Markdown. Every code block labelled with its file path, or with where it runs (WSL, node0)
  and whether it is read-only.
- After editing files, show `git diff --stat` and propose a commit message.
- Default under ~400 words. End with the single next action the owner should take.
