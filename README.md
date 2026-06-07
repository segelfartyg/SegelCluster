# SegelCluster

GitOps configuration for **segel-cluster**, a personal Kubernetes cluster reconciled by [Flux CD](https://fluxcd.io/).

Flux watches this repository (`https://github.com/segelfartyg/SegelCluster`, branch `main`) and applies everything under [`clusters/segel-cluster`](clusters/segel-cluster), pruning resources that are removed.

## Repository layout

```
clusters/segel-cluster/
├── flux-system/   # Flux bootstrap manifests (GitRepository + Kustomization, gotk components)
├── namespaces/    # Namespace definitions (monitoring, traefik)
├── monitoring/    # Observability stack and the SMS application stack
└── traefik/       # Ingress / Gateway API / external exposure
```

## What's deployed

### Observability (`monitoring/`)
- **kube-prometheus-stack** — Prometheus, Alertmanager and Grafana for metrics
- **Loki** (monolithic deployment mode, S3-backed via the `loki-s3` secret) — log storage with 28-day retention
- **Grafana Alloy** (DaemonSet) — collects pod logs and Kubernetes cluster events and ships them to Loki
- **Headlamp** — web UI for the Kubernetes API

### SMS application stack (sourced from [segelfartyg/sms](https://github.com/segelfartyg/sms))
- **sms-backend** — backend API (DB connection from `sms-backend-db` secret)
- **sms-warehouse** — data warehouse service (DB connection from `sms-warehouse-db` secret)
- **sms-k8s-exporter** — exports cluster info to the warehouse
- **sms-page-viewer** — frontend, exposed externally via a Traefik `HTTPRoute`

### Ingress & networking (`traefik/`)
- **Gateway API CRDs** — installed via a Flux `Kustomization` pointing at the upstream `kubernetes-sigs/gateway-api` repo
- **Traefik** — ingress controller running as the Gateway API implementation, exposed as a `NodePort` service
- **cloudflared** — Cloudflare Tunnel deployment (2 replicas) that exposes services to the internet without opening inbound ports (tunnel token from the `tunnel-token` secret)

## External infrastructure (not in this repo)

Some dependencies run outside the Kubernetes cluster, on separate VMs, and are consumed via the secrets below:
- **PostgreSQL** — databases for `sms-backend` and `sms-warehouse` (connected to via the `sms-backend-db` / `sms-warehouse-db` secrets' `DATABASE_URL`)
- **MinIO** — S3-compatible object storage backing Loki's chunk/ruler/admin buckets (connected to via the `loki-s3` secret's endpoint/credentials)

## Secrets

Several `HelmRelease`s and resources reference Kubernetes Secrets that are **not** stored in this repo and must be created in the cluster out of band:

| Secret | Namespace | Used by |
| --- | --- | --- |
| `loki-s3` | monitoring | Loki S3 storage endpoint/credentials |
| `sms-backend-db` | monitoring | sms-backend `DATABASE_URL` |
| `sms-warehouse-db` | monitoring | sms-warehouse `DATABASE_URL` |
| `tunnel-token` | traefik | cloudflared tunnel token |
| `flux-system` | flux-system | Git repository auth for Flux |

## Bootstrapping

Flux is bootstrapped against this repository with:
- Source: `https://github.com/segelfartyg/SegelCluster.git`, branch `main`
- Sync path: `./clusters/segel-cluster`
- Reconciliation interval: 10 minutes
