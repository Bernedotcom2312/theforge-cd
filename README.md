# theforge-cd

GitOps repository for ArgoCD, based on the **app-of-apps** pattern. This repo only contains Kubernetes/ArgoCD manifests (YAML): no build, lint, or test tooling. "Correctness" simply means valid YAML that ArgoCD can sync.

## Architecture

- [root-app.yaml](root-app.yaml) — the root ArgoCD `Application`, applied manually once to bootstrap the cluster. It points at the [apps/](apps/) directory (`repoURL` + `path: apps`) and auto-syncs (prune + selfHeal) everything it finds there.
- [apps/*.yaml](apps/) — one ArgoCD `Application` manifest per component. Each either:
  - pulls a Helm chart directly from a chart repo (`source.chart` + `source.repoURL` pointing at a Helm repo, with inline `helm.values`), e.g. [cert-manager.yaml](apps/cert-manager.yaml), [alloy.yaml](apps/alloy.yaml), [loki.yaml](apps/loki.yaml), [kube-prometheus-stack.yaml](apps/kube-prometheus-stack.yaml), [nginx-gateway.yaml](apps/nginx-gateway.yaml); or
  - points at a `path` in this git repo, e.g. [nginx.yaml](apps/nginx.yaml) → `path: nginx`.
- [nginx/](nginx/) — plain Kubernetes manifests (Namespace, Deployment, Service, ReferenceGrant) for a demo app deployed into the `demo` namespace via [apps/nginx.yaml](apps/nginx.yaml).

## Sync ordering (sync-waves)

Applications declare `argocd.argoproj.io/sync-wave` annotations to sequence installation, since later components depend on earlier ones (e.g. Gateway API CRDs and cert-manager CRDs must exist before anything that consumes them):

| Wave | Apps | Notes |
|---|---|---|
| 0 | `gateway-api-crds`, `cert-manager` | CRDs / cluster-wide prerequisites |
| 1 | `nginx-gateway` | Gateway API implementation, depends on wave 0 CRDs |
| 2 | `loki` | log storage backend |
| 3 | `alloy`, `kube-prometheus-stack` | observability agents/stack, Alloy ships logs to Loki (wave 2) |
| 4 | `nginx` (demo app) | deployed into `demo` namespace, routed via the gateway |

When adding a new `Application`, set its sync-wave relative to what it depends on (CRDs/controllers before consumers), not just append to the end.

## Conventions shared by every Application manifest

- `finalizers: [resources-finalizer.argocd.argoproj.io]` so deleting the Application also cleans up its resources.
- `syncPolicy.automated: { prune: true, selfHeal: true }` — cluster state is expected to always match git.
- `syncOptions: [CreateNamespace=true]` when the target namespace doesn't already exist, and `ServerSideApply=true` for Helm-sourced apps (needed for large CRDs).
- Identical retry/backoff block: `limit: 5`, `duration: 5s`, `factor: 2`, `maxDuration: 3m`.
- `destination.server` is always the in-cluster API (`https://kubernetes.default.svc`); this repo only targets the local cluster.

## Bootstrap

Initial deployment is done by manually applying `root-app.yaml`:

```bash
kubectl apply -f root-app.yaml
```

ArgoCD then takes over and automatically syncs everything present in [apps/](apps/).

## Working in this repo

- New components go in [apps/](apps/) as a new `Application` manifest (Helm chart or path-based); resources for path-based apps live in a same-named top-level directory (see [nginx/](nginx/) as the pattern to follow).
- Because `syncPolicy.automated.selfHeal` is on everywhere, manual `kubectl edit`/`kubectl apply` changes against these resources will be reverted by ArgoCD — all changes must go through this git repo.
- There's no local way to "run" this repo; validate by checking YAML is well-formed and, if possible, diffing against ArgoCD (`argocd app diff <name>`) or `kubectl apply --dry-run=server -f <file>` before committing.
