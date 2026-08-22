# DocuMind GitOps

Kubernetes manifests and ArgoCD `Application` definitions for DocuMind.
ArgoCD watches this repo and applies whatever's here - it's the single
source of truth for what should be running in the cluster. Part of the
[DocuMind platform](https://github.com/qezman/DocuMind-Infrastructure); see that
repo's README for the full system overview and architecture.

## How it's used

Nobody edits this repo's manifests to deploy a new version - CI in the
app repos (`documind-frontend`, `documind-backend`, `documind-agent`) does
that automatically, patching the relevant image tag on every push to
`main`. Manual edits here are for actual config changes (a new env var, a
new policy, a resource limit), not routine deploys. ArgoCD's `selfHeal` is
on, so a manual `kubectl edit` against the live cluster gets reverted back
to match this repo on the next sync.

## Layout

- **apps/** - one ArgoCD `Application` per component (`frontend`,
`backend`, `agent`, `cluster`), each pointing at its own subfolder under
`manifests/`
- **manifests/frontend/** - `rollout.yaml` (canary), `service.yaml`,
`ingress.yaml`, `namespace.yaml` (owns the `documind` namespace - see
note below), `grafana-proxy.yaml`
- **manifests/backend/** - `rollout.yaml` (canary), `service.yaml`,
`serviceaccount.yaml` (IRSA role), `externalsecret.yaml`,
`secretstore.yaml`, `alerts.yaml`
- **manifests/agent/** - `deployment.yaml` (plain Deployment, not a
canary), `service.yaml`, `rbac.yaml` (the read-only role),
`externalsecret.yaml`
- **manifests/cluster/** - Kyverno policies (`policy-disallow-root.yaml`,
`policy-restrict-registries.yaml`), applied cluster-wide



## One thing worth knowing

The `documind` namespace is deliberately declared in exactly one place -
`manifests/frontend/namespace.yaml`. It used to also get created via a
duplicate `Namespace` block, which caused ArgoCD's `backend` and `frontend`
Applications to fight over ownership of the same object
(`SharedResourceWarning`, apps flapping between `Synced`/`OutOfSync`). If
you add a new component here, don't give it its own `Namespace` object or
`CreateNamespace=true` sync option - `frontend` already owns it.