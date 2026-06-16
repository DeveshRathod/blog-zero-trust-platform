# zero-trust-k8s

Kubernetes manifests for the **Zero-Trust Microservices Blog Platform** — a
production-style multi-service platform deployed on a local kind cluster with
Istio ambient mode, ArgoCD GitOps, and a full observability stack.

## Architecture

```
                        ┌─────────────────────────────────────────┐
                        │  kind cluster (zero-trust)              │
                        │                                         │
  localhost:80  ──────► │  Istio IngressGateway                  │
                        │          │                              │
                        │    ┌─────▼──────┐                      │
                        │    │ zt-gateway │  api-gateway         │
                        │    └─────┬──────┘                      │
                        │          │                              │
                        │    ┌─────┴──────┐                      │
                        │    ▼            ▼                       │
                        │ zt-auth      zt-posts                   │
                        │ auth-svc     blog-svc                   │
                        │    │            │    └──► zt-cache      │
                        │    │            └──────► zt-posts-db    │
                        │    └──► zt-auth-db                      │
                        │    └──► zt-email (internal only)        │
                        │                                         │
                        │  zt-web  blog-frontend (nginx)          │
                        └─────────────────────────────────────────┘

  ztunnel (DaemonSet) — automatic mTLS on all cross-namespace traffic
  waypoint proxies    — L7 AuthorizationPolicy, retries, circuit breaking
```

## Namespaces

| Namespace | Service | Stack |
|---|---|---|
| `zt-gateway` | api-gateway | Express, reverse proxy |
| `zt-auth` | auth-service | Express, Postgres |
| `zt-auth-db` | Postgres 16 | — |
| `zt-posts` | blog-service | Express, MongoDB, Redis |
| `zt-posts-db` | MongoDB 7 | — |
| `zt-cache` | Redis 7 | LRU cache-aside |
| `zt-email` | email-service | Express, nodemailer |
| `zt-web` | blog-frontend | React, Vite, nginx |

## Repo structure

```
zero-trust-k8s/
├── argocd/
│   ├── project.yaml          ← AppProject scoping all apps to zt-* namespaces
│   └── apps/                 ← one ArgoCD Application per service
│       ├── gateway.yaml
│       ├── auth.yaml
│       └── ...
└── apps/                     ← ArgoCD watches this — deployment + service per service
    ├── gateway/
    ├── auth/
    ├── auth-db/
    ├── posts/
    ├── posts-db/
    ├── cache/
    ├── email/
    └── web/
```

## Prerequisites

Run `prerequisite.sh` (not in this repo — contains secrets) once on a fresh machine:

```bash
# 1. Fill in your values at the top of prerequisite.sh
# 2. Run it
chmod +x prerequisite.sh
./prerequisite.sh
```

This script:
- Creates a kind cluster with port 80/443 mapped to Istio's IngressGateway
- Installs Istio in **ambient mode** (ztunnel + waypoint proxies, no sidecars)
- Installs Prometheus, Grafana, Jaeger, Kiali
- Creates all 8 `zt-*` namespaces with `istio.io/dataplane-mode=ambient`
- Deploys waypoint proxies for L7 in each app namespace
- Installs ArgoCD
- Creates K8s Secrets per namespace (JWT, DB passwords, SMTP credentials)
- Applies `argocd/project.yaml` and `argocd/apps/` — ArgoCD takes over from there

## Deploying

After `prerequisite.sh` runs, ArgoCD auto-syncs everything from this repo.

```bash
# Watch sync progress
kubectl get applications -n argocd -w

# Access ArgoCD UI
kubectl port-forward svc/argocd-server -n argocd 8080:443
# https://localhost:8080  (admin / password printed by prerequisite.sh)

# Test before Istio routing is set up
kubectl port-forward svc/api-gateway 9090:8080 -n zt-gateway
curl http://localhost:9090/healthz
```

## Observability

```bash
istioctl dashboard kiali      # service graph + mTLS topology
istioctl dashboard grafana    # golden signal dashboards
istioctl dashboard jaeger     # distributed traces
istioctl dashboard prometheus # raw metrics
```

## Verify ambient mode

```bash
# No istio-proxy sidecar in any pod (that's the point of ambient mode)
kubectl get pods -n zt-auth -o jsonpath='{.items[*].spec.containers[*].name}'

# ztunnel running on every node
kubectl get pods -n istio-system -l app=ztunnel

# Waypoint proxy per namespace
kubectl get waypoints -A

# ztunnel routing table
istioctl ztunnel-config all
```

## Chaos engineering

Inject failures into blog-service to test Istio resilience policies:

```bash
kubectl set env deployment/blog-service \
  CHAOS_ERROR_RATE=0.3 \
  CHAOS_LATENCY_MS=1000 \
  -n zt-posts
```

Watch the effect in Kiali's service graph and Grafana's error rate panels.
Reset with `CHAOS_ERROR_RATE=0 CHAOS_LATENCY_MS=0`.

## What's next

- `apps/*/istio/` — AuthorizationPolicy, VirtualService, DestinationRule per service
- Istio Gateway + VirtualService for external access via `localhost`
- HorizontalPodAutoscaler on gateway and posts services
