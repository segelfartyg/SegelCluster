# SegelCluster

GitOps configuration for **segel-cluster**, a personal Kubernetes cluster reconciled by [Flux CD](https://fluxcd.io/).

Flux watches this repository (`https://github.com/segelfartyg/SegelCluster`, branch `main`) and applies everything under [`clusters/segel-cluster`](clusters/segel-cluster), pruning resources that are removed.

## Repository layout

```
clusters/segel-cluster/
├── flux-system/   # Flux bootstrap manifests (GitRepository + Kustomization, gotk components)
├── namespaces/    # Namespace definitions (fiskekartan, monitoring, traefik)
├── fiskekartan/   # Fiskekartan application
├── monitoring/    # Observability stack and the SMS application stack
└── traefik/       # Ingress / Gateway API / Keycloak / external exposure
```

## What's deployed

### Observability (`monitoring/`)
- **kube-prometheus-stack** — Prometheus, Alertmanager and Grafana for metrics. Grafana is exposed via a Traefik `HTTPRoute` on `grafana.swaren.se` and authenticates against the `segel-cluster` realm in Keycloak via generic OAuth with PKCE (client credentials from the `grafana-oauth` secret). Grafana roles are mapped from the Keycloak `grafana-oauth` client roles: `admin` → Grafana Admin, `editor` → Editor, everyone else → Viewer. Loki is pre-provisioned as a datasource
- **Loki** (monolithic deployment mode, S3-backed via the `loki-s3` secret) — log storage with 28-day retention enforced by the compactor. The chart is pinned to `17.3.1` from the `grafana-community` Helm repo (a 17.4.x upgrade timed out, so bump it separately)
- **Grafana Alloy** (DaemonSet) — collects pod logs and Kubernetes cluster events and ships them to Loki
- **Headlamp** — web UI for the Kubernetes API

### SMS application stack (sourced from [segelfartyg/sms](https://github.com/segelfartyg/sms))
- **sms-backend** — backend API (DB connection from `sms-backend-db` secret)
- **sms-warehouse** — data warehouse service (DB connection from `sms-warehouse-db` secret)
- **sms-k8s-exporter** — exports cluster info to the warehouse
- **sms-page-viewer** — frontend, exposed via a Traefik `HTTPRoute` on `lab.swaren.se`

### Fiskekartan (`fiskekartan/`, sourced from [segelfartyg/fiskekartan](https://github.com/segelfartyg/fiskekartan))
- **fiskekartan** — Helm chart deployed from the `main` branch into the `fiskekartan` namespace (`reconcileStrategy: Revision`, so every new commit triggers an upgrade), exposed via a Traefik `HTTPRoute` on `fiskekartan.swaren.se`
- Authenticates users with OIDC against the `segel-cluster` Keycloak realm (audience `fiskekartan`)
- Stores data in PostgreSQL and catch photos in the `fiskekartan-images` RustFS bucket; connection details come from the `fiskekartan-secrets` secret. `CATCH_PHOTO_SIGNING_KEY` is optional — without it the app generates a random key on each start, which breaks existing photo links after a restart

### Ingress & networking (`traefik/`)
- **Gateway API CRDs** — standard-channel CRDs installed via a Flux `Kustomization` pointing at the upstream `kubernetes-sigs/gateway-api` repo (tag `v1.5.1`)
- **Traefik** — ingress controller running as the Gateway API implementation (`traefik-gateway`, accepting routes from all namespaces), exposed as a `NodePort` service. The `web` entrypoint trusts `X-Forwarded-*` headers (`forwardedHeaders.insecure`) so the original scheme/host from Cloudflare reaches the backends
- **cloudflared** — Cloudflare Tunnel deployment (2 replicas) that exposes services to the internet without opening inbound ports (tunnel token from the `tunnel-token` secret)
- **Keycloak** (`26.6.3`) — identity/auth provider ([getting-started-kube](https://www.keycloak.org/getting-started/getting-started-kube)-based StatefulSet, 2 replicas, clustered via Infinispan/JGroups over the headless `keycloak-discovery` service), exposed via a Traefik `HTTPRoute` on the `auth.swaren.se` hostname. Admin login from the `keycloak-admin` secret, database connection from the `keycloak-db` secret. The public hostname is pinned via `KC_HOSTNAME=https://auth.swaren.se` so the admin console generates correct URLs through the Cloudflare Tunnel → Traefik path.

## External infrastructure (not in this repo)

Some dependencies run outside the Kubernetes cluster, on separate VMs, and are consumed via the secrets below:
- **PostgreSQL** — databases for `sms-backend`, `sms-warehouse`, `keycloak` and `fiskekartan` (connected to via the `sms-backend-db` / `sms-warehouse-db` / `keycloak-db` / `fiskekartan-secrets` secrets)
- **[RustFS](https://rustfs.com/)** — S3-compatible object storage backing Loki's chunk/ruler/admin buckets (via the `loki-s3` secret) and Fiskekartan's `fiskekartan-images` bucket (via `fiskekartan-secrets`)

## Secrets

Several resources reference Kubernetes Secrets that are **not** stored in this repo and must be created in the cluster out of band:

| Secret | Namespace | Used by | Keys |
| --- | --- | --- | --- |
| `loki-s3` | monitoring | Loki S3 storage endpoint/credentials | `endpoint`, `accessKeyId`, `secretAccessKey` |
| `grafana-oauth` | monitoring | Grafana Keycloak generic OAuth client credentials | `client-id`, `client-secret` |
| `sms-backend-db` | monitoring | sms-backend `DATABASE_URL` | `DATABASE_URL` |
| `sms-warehouse-db` | monitoring | sms-warehouse `DATABASE_URL` | `DATABASE_URL` |
| `fiskekartan-secrets` | fiskekartan | Fiskekartan DB, RustFS (S3) and photo-link signing | `DATABASE_URL`, `MINIO_ENDPOINT`, `MINIO_ACCESS_KEY`, `MINIO_SECRET_KEY`, `CATCH_PHOTO_SIGNING_KEY` (optional) |
| `tunnel-token` | traefik | cloudflared tunnel token | `token` |
| `keycloak-admin` | traefik | Keycloak admin console login | `username`, `password` |
| `keycloak-db` | traefik | Keycloak Postgres connection | `host`, `port`, `database`, `username`, `password` |
| `flux-system` | flux-system | Git repository auth for Flux | - |

## Bootstrapping

Flux is bootstrapped against this repository with:
- Source: `https://github.com/segelfartyg/SegelCluster.git`, branch `main`
- Sync path: `./clusters/segel-cluster`
- Source polling interval: 1 minute; Kustomization reconciliation interval: 10 minutes
