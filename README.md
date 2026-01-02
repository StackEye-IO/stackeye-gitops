# StackEye GitOps

GitOps repository for StackEye Kubernetes deployments, managed by ArgoCD.

## Architecture

```
                    ┌─────────────────────────────────┐
                    │       Cloudflare Pages          │
                    │     (Web Frontend - Free)       │
                    │      app.stackeye.io            │
                    └───────────────┬─────────────────┘
                                    │ API calls
                                    ▼
┌───────────────────────────────────────────────────────────────────┐
│                     a1-ops-prd (On-Prem - Free)                   │
│  ┌─────────────┐  ┌─────────────┐  ┌─────────────┐  ┌──────────┐ │
│  │   ArgoCD    │  │  API Server │  │    CNPG     │  │  Valkey  │ │
│  │ (multi-     │  │ api.stack-  │  │ PostgreSQL  │  │  Cache   │ │
│  │  cluster)   │  │  eye.io     │  │             │  │          │ │
│  └──────┬──────┘  └─────────────┘  └─────────────┘  └──────────┘ │
│         │ manages                                                 │
└─────────┼─────────────────────────────────────────────────────────┘
          │
    ┌─────┴─────────────────────────────┐
    │                                   │
    ▼                                   ▼
┌─────────────────────┐     ┌─────────────────────┐
│  stackeye-nyc3      │     │  stackeye-sfo3      │
│  DOKS ($64/mo)      │     │  DOKS ($64/mo)      │
│  ┌───────────────┐  │     │  ┌───────────────┐  │
│  │    Workers    │  │     │  │    Workers    │  │
│  │  (East Coast) │  │     │  │  (West Coast) │  │
│  │   region=nyc3 │  │     │  │   region=sfo3 │  │
│  └───────────────┘  │     │  └───────────────┘  │
└─────────────────────┘     └─────────────────────┘

Total Monthly Cost: ~$128/mo
```

## Repository Structure

```
stackeye-gitops/
├── apps/
│   ├── project.yaml              # ArgoCD AppProject
│   ├── stackeye-api.yaml         # API → on-prem (dev, staging, prod)
│   ├── stackeye-workers-nyc3.yaml # Workers → DOKS NYC3
│   └── stackeye-workers-sfo3.yaml # Workers → DOKS SFO3
├── argocd/
│   └── values.yaml               # ArgoCD installation config
├── environments/
│   ├── dev/
│   │   └── values.yaml           # Dev image tags
│   ├── staging/
│   │   └── values.yaml           # Staging image tags
│   └── prod/
│       └── values.yaml           # Prod image tags
└── README.md
```

## Cluster Topology

| Cluster | Purpose | Cost |
|---------|---------|------|
| a1-ops-prd (on-prem) | API, ArgoCD, PostgreSQL, Valkey | Free |
| stackeye-nyc3 (DOKS) | Workers - East Coast | $64/mo |
| stackeye-sfo3 (DOKS) | Workers - West Coast | $64/mo |

**Web Frontend**: Deployed to Cloudflare Pages (Free)

## Network Connectivity

| Source | Destination | Method |
|--------|-------------|--------|
| Cloudflare Pages | API (a1-ops-prd) | Public HTTPS (api.stackeye.io) |
| Workers (NYC3) | PostgreSQL (a1-ops-prd) | Tailscale VPN |
| Workers (SFO3) | PostgreSQL (a1-ops-prd) | Tailscale VPN |
| ArgoCD (a1-ops-prd) | DOKS clusters | ServiceAccount tokens |

## Tag Convention

| Tag Pattern | Target Environment | Description |
|-------------|-------------------|-------------|
| `v1.2.3` | dev | Auto-deploys to development |
| `v1.2.3-staging` | staging | Deploys to staging |
| `v1.2.3-prod` | prod | Deploys to production |

## Prerequisites

- Kubernetes 1.28+
- Helm 3.14+
- ArgoCD 2.10+
- Tailscale installed on DOKS worker nodes

## ArgoCD Installation

```bash
# Add ArgoCD Helm repo
helm repo add argo https://argoproj.github.io/argo-helm
helm repo update

# Install ArgoCD on a1-ops-prd
helm install argocd argo/argo-cd -n argocd --create-namespace \
  -f argocd/values.yaml

# Wait for ArgoCD to be ready
kubectl wait --for=condition=available deployment/argocd-server -n argocd --timeout=300s

# Get initial admin password
kubectl -n argocd get secret argocd-initial-admin-secret -o jsonpath="{.data.password}" | base64 -d
```

## Add Remote Clusters

After ArgoCD is installed, add the DOKS clusters:

```bash
# Login to ArgoCD
argocd login argocd.stackeye.io

# Add DOKS clusters (uses kubeconfig context)
argocd cluster add stackeye-nyc3 --kubeconfig ~/.kube/mattox/stackeye-nyc3
argocd cluster add stackeye-sfo3 --kubeconfig ~/.kube/mattox/stackeye-sfo3

# Verify clusters
argocd cluster list
```

## Bootstrap Applications

After ArgoCD is installed and clusters are added:

```bash
# Create the StackEye project
kubectl apply -f apps/project.yaml

# Create all applications
kubectl apply -f apps/stackeye-api.yaml
kubectl apply -f apps/stackeye-workers-nyc3.yaml
kubectl apply -f apps/stackeye-workers-sfo3.yaml
```

## Deployment Workflow

### Deploy to Dev (automatic on any v*.*.* tag)

```bash
cd stackeye/
git tag v1.2.3
git push origin v1.2.3
```

### Promote to Staging

```bash
git tag v1.2.3-staging
git push origin v1.2.3-staging
```

### Promote to Production

```bash
git tag v1.2.3-prod
git push origin v1.2.3-prod
```

## Manual Operations

### Force Sync

```bash
argocd app sync stackeye-api-prod
argocd app sync stackeye-worker-nyc3-prod
argocd app sync stackeye-worker-sfo3-prod
```

### Rollback

```bash
argocd app rollback stackeye-api-prod
```

### View Application Status

```bash
argocd app list
argocd app get stackeye-api-prod
argocd app get stackeye-worker-nyc3-prod
argocd app get stackeye-worker-sfo3-prod
```

## Related Repositories

| Repository | Purpose |
|------------|---------|
| [stackeye](https://github.com/StackEye-IO/stackeye) | Backend API + Worker |
| [stackeye-web](https://github.com/StackEye-IO/stackeye-web) | Frontend (Cloudflare Pages) |
| [stackeye-deploy](https://github.com/StackEye-IO/stackeye-deploy) | Helm charts |

## License

Apache 2.0
