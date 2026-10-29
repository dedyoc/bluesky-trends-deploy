# interview-notes.md — what I can explain, slice by slice

One entry per working slice, written right after it works. Answers are short and in my own
words; if I can't say it in two sentences, I don't understand it yet. Record real numbers from
node0 (MiB, events/s, seconds), not textbook ones.

Format:

```
## <Component or slice>
- What it does here: …
- Why this, not <obvious alternative>: …
- Trade-off I accepted: …
- What broke, and how I found it: …
- Numbers: …
- Likely questions:
  - Q: … A: …
```

---

## Multi-tenancy on one cluster: AppProject + quota (platform, before slice 0)

- What it does here: the platform repo (`homelab`) owns the `bsky` Namespace, a restricting
  ArgoCD `AppProject`, the `Application`, and a `ResourceQuota`. This repo can only deploy
  namespaced objects into `bsky`.
- Why this, not one repo for everything: a tenant mistake can't touch the platform or other
  tenants. ArgoCD refuses a cluster-scoped kind or a foreign namespace from this repo (tested
  with a throwaway tenant, homelab ADR-034).
- Why operators live in the platform repo: an operator installs CRDs and ClusterRoles, which
  are cluster-wide. Platform teams usually own operators and tenants submit custom resources
  (`Kafka`, `FlinkDeployment`): the same split as "the DBA runs Postgres, the app team creates
  tables".
- Trade-off I accepted: every new operator is a commit in a second repo before I can use its CRs.
- Likely questions:
  - Q: Why a ResourceQuota on a single-node homelab? A: RAM is the scarce resource. The quota
    turns "the box is slow" into a clear "pod refused, quota exceeded" error, and the tenant
    can't raise it, because the AppProject blacklists the kind.
  - Q: What happens if you delete the tenant directory in the platform repo? A: ArgoCD prunes
    the Namespace, and Kubernetes deletes everything in it, PVCs included. That's the
    offboarding path, and also the biggest footgun.
