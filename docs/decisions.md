# decisions.md — ADR log (tenant `bsky`)

**Append-only.** New entries go at the bottom with the next number, and get a row in the
Index. To change a decision, write a superseding ADR or a dated **Addendum** under the
original; never edit the original text. "Reason not recorded" is not a justification.

Platform decisions are cited as "homelab ADR-0NN" and live only in `~/homelab/docs/decisions.md`.
The ones that frame this tenant: homelab ADR-031 (tenants in their own repos), ADR-034
(onboarding under `apps/tenants/`), ADR-042 (this tenant and repo split), ADR-043 (operators
platform-owned, memory quota).

## Index

| # | Date | Decision | Status |
|---|---|---|---|
| 001 | 2026-10-28 | Flat Kustomize layout: root lists one directory per component | Active |

---

### ADR-001: Flat Kustomize layout: root lists one directory per component
- *Context:* the Application points at the repo root. The tenant will hold ~10 components
  across five slices, some hand-written, some rendered from Helm.
- *Decision:* **A root `kustomization.yaml` sets `namespace: bsky` and lists one directory per
  component (`kafka/`, `ingest/`, `clickhouse/`, …). Each directory has its own
  `kustomization.yaml`. Helm charts are rendered with `helm template` at a pinned version into
  `<component>/vendor/`, with the values file beside them.**
- *Reason:* same pattern as homelab's `apps/` and the `nook` tenant (homelab ADR-036 addendum),
  so there's one layout to learn. The `AppProject` allows only this repo as a source, so charts
  can't be pulled at sync time.
- *Alternatives:* plain YAML with directory recursion (applies any stray `.yaml`; nothing sets
  the namespace); overlays per environment (there is one environment).
- *Consequences:* adding a component is a directory plus one line in the root list. Removing a
  line prunes everything in that directory, so a stateful one loses its data.
- *Date:* 2026-10-28
