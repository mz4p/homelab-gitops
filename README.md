# ☸️ homelab-gitops

GitOps state for a small hybrid Kubernetes homelab. Everything that runs in the
cluster is described here and reconciled by ArgoCD — no manual `kubectl apply`
after bootstrap.

The cluster itself (OS prep, K3s install, Tailscale join) is provisioned by
[homelab-infra](https://github.com/mz4p/homelab-infra). This repository starts
where that one ends.

## Architecture

```
        git push
           │
           ▼
   ┌───────────────┐        watches        ┌──────────────────────────┐
   │  this repo    │ ────────────────────► │  ArgoCD (App-of-Apps)    │
   └───────────────┘                       │  root app: bootstrap.yaml│
                                           └────────────┬─────────────┘
                                                        │ syncs
                        ┌───────────────────────────────┼───────────────────────────┐
                        ▼                               ▼                           ▼
              infrastructure/                     apps/monitoring/             apps/homarr/
              ingress, TLS                    kube-prometheus-stack            dashboard
```

- **App-of-Apps** — `bootstrap.yaml` is the only object applied by hand. It points
  ArgoCD at `apps/`, and every `Application` found there is created and synced
  automatically.
- **Two application layers** — platform components (`infrastructure/`) are kept
  separate from workloads (`apps/`), so ingress or networking changes do not
  redeploy monitoring and dashboards.
- **Helm values as files** — chart configuration lives in `values/<app>/values.yaml`
  instead of `--set` flags, so every change is reviewable in a diff.

## Repository layout

| Path | What it contains |
| :--- | :--- |
| `bootstrap.yaml` | Root ArgoCD `Application` (App-of-Apps entry point) |
| `apps/infrastructure.yaml` | ArgoCD app syncing `infrastructure/` (recurse) |
| `apps/monitoring/kube-prometheus-stack.yaml` | Prometheus + Grafana + Alertmanager |
| `apps/homarr/homarr.yaml` | Homarr start-page dashboard |
| `infrastructure/ingress.yaml` | Cluster ingress resources |
| `infrastructure/argocd-ingress.yaml` | Ingress exposing the ArgoCD UI |
| `values/monitoring/values.yaml` | kube-prometheus-stack values |
| `values/homarr/values.yaml` | Homarr values |

## Cluster

Three nodes, mixed architecture, joined over Tailscale:

| Node | Hardware | OS | Arch |
| :--- | :--- | :--- | :--- |
| master | HP EliteDesk, 16 GB RAM | Ubuntu Server 24.04 | `amd64` |
| worker | Raspberry Pi 5, 4 GB | Raspberry Pi OS Lite | `arm64` |
| worker | Raspberry Pi 4B, 4 GB | Raspberry Pi OS Lite | `arm64` |

Running mixed `amd64`/`arm64` nodes means every workload has to be multi-arch
aware — that constraint is the main reason chart selection here is deliberate.

## Bootstrap

```bash
# 1. the cluster must already exist (see homelab-infra)
kubectl get nodes

# 2. apply the root application once
kubectl apply -f bootstrap.yaml

# 3. watch ArgoCD take over
kubectl -n argocd get applications
```

After step 2, anything merged to `main` is reconciled automatically
(`automated.prune: true`, `selfHeal: true`).

## Secrets

No credentials are stored in this repository. Components that need them reference
existing Kubernetes Secrets created out-of-band — for example Grafana admin
credentials are wired through `existingSecret: grafana-admin-credentials`.

## What I'd add next

- Sealed Secrets or SOPS so more of the stack can live in Git
- A staging namespace to exercise promotion between overlays
- Alerting rules routed to a real notification channel
- A small Go CLI for repetitive cluster tasks (personal Go learning project)
