# Tailscale Kubernetes Operator Setup

This document describes how to install Tailscale on the DOKS clusters (stackeye-nyc3 and stackeye-sfo3) to enable secure connectivity to PostgreSQL on a1-ops-prd.

## Architecture

```
┌─────────────────────────────────────────────────────────────────────┐
│                     a1-ops-prd (On-Prem)                            │
│  ┌─────────────────┐       ┌─────────────────┐                     │
│  │   PostgreSQL    │       │     Valkey      │                     │
│  │ 100.64.x.x:5432 │       │ 100.64.x.x:6379 │                     │
│  └────────┬────────┘       └────────┬────────┘                     │
│           │                         │                               │
│           └─────────┬───────────────┘                               │
│                     │ Tailscale                                     │
└─────────────────────┼───────────────────────────────────────────────┘
                      │
        ┌─────────────┴─────────────┐
        │                           │
        ▼                           ▼
┌───────────────────┐     ┌───────────────────┐
│   stackeye-nyc3   │     │   stackeye-sfo3   │
│   ┌───────────┐   │     │   ┌───────────┐   │
│   │ Tailscale │   │     │   │ Tailscale │   │
│   │ Operator  │   │     │   │ Operator  │   │
│   └─────┬─────┘   │     │   └─────┬─────┘   │
│         │         │     │         │         │
│   ┌─────▼─────┐   │     │   ┌─────▼─────┐   │
│   │  Workers  │◄──┼─────┼───│  Workers  │   │
│   └───────────┘   │     │   └───────────┘   │
└───────────────────┘     └───────────────────┘
```

## Prerequisites

1. **Tailscale Account** with admin access
2. **Tailnet Policy File** configured with required tags
3. **OAuth credentials** for Kubernetes operator

## Step 1: Configure Tailscale ACL Tags

Add these tags to your Tailscale ACL policy file at https://login.tailscale.com/admin/acls:

```json
{
  "tagOwners": {
    "tag:k8s-operator": ["autogroup:admin"],
    "tag:k8s": ["tag:k8s-operator"],
    "tag:stackeye-worker": ["tag:k8s-operator"]
  },
  "acls": [
    {
      "action": "accept",
      "src": ["tag:stackeye-worker"],
      "dst": ["tag:stackeye-db:5432", "tag:stackeye-cache:6379"]
    }
  ]
}
```

## Step 2: Generate OAuth Credentials

1. Go to https://login.tailscale.com/admin/settings/oauth
2. Click "Generate OAuth client"
3. Configure:
   - **Description**: `stackeye-nyc3-k8s-operator`
   - **Scopes**:
     - `Devices Core` (Write)
     - `Auth Keys` (Write)
     - `Services` (Write)
   - **Tags**: `tag:k8s-operator`
4. Copy the **Client ID** and **Client Secret**
5. **Repeat for SFO3 cluster** with description `stackeye-sfo3-k8s-operator`

## Step 3: Install Tailscale Operator on NYC3

```bash
# Add Tailscale Helm repository
helm repo add tailscale https://pkgs.tailscale.com/helmcharts
helm repo update

# Set your kubeconfig
export KUBECONFIG=~/.kube/mattox/stackeye-nyc3

# Install Tailscale operator (replace with your credentials)
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  --namespace=tailscale \
  --create-namespace \
  --set-string oauth.clientId="<NYC3_OAUTH_CLIENT_ID>" \
  --set-string oauth.clientSecret="<NYC3_OAUTH_CLIENT_SECRET>" \
  --wait

# Verify installation
kubectl get pods -n tailscale
```

## Step 4: Install Tailscale Operator on SFO3

```bash
# Set your kubeconfig
export KUBECONFIG=~/.kube/mattox/stackeye-sfo3

# Install Tailscale operator (replace with your credentials)
helm upgrade --install tailscale-operator tailscale/tailscale-operator \
  --namespace=tailscale \
  --create-namespace \
  --set-string oauth.clientId="<SFO3_OAUTH_CLIENT_ID>" \
  --set-string oauth.clientSecret="<SFO3_OAUTH_CLIENT_SECRET>" \
  --wait

# Verify installation
kubectl get pods -n tailscale
```

## Step 5: Verify Tailscale Connectivity

After installation, verify nodes appear in Tailscale admin:

1. Go to https://login.tailscale.com/admin/machines
2. Look for nodes with names like:
   - `tailscale-operator-stackeye-nyc3`
   - `tailscale-operator-stackeye-sfo3`

## Step 6: Configure Worker Pods for Tailscale Access

Option A: Use Tailscale sidecar (recommended for pod-level access):

```yaml
# In stackeye-worker values.yaml
worker:
  tailscale:
    enabled: true
    authKeySecretName: tailscale-auth-key
```

Option B: Use cluster-wide egress (for all pods in cluster):

Create a Connector resource:

```yaml
apiVersion: tailscale.com/v1alpha1
kind: Connector
metadata:
  name: stackeye-egress
  namespace: tailscale
spec:
  hostname: stackeye-nyc3-egress  # or stackeye-sfo3-egress
  subnetRouter:
    advertiseRoutes:
      - 10.0.0.0/8  # Your tailnet range
```

## Step 7: Update Worker Configuration

Update the worker Helm values to use Tailscale DNS for database connection:

```yaml
# environments/prod/values.yaml
worker:
  database:
    host: "a1-ops-prd.tailnet-xxxx.ts.net"  # Use Tailscale MagicDNS
    port: 5432
  redis:
    host: "a1-ops-prd.tailnet-xxxx.ts.net"
    port: 6379
```

## Verification Commands

```bash
# Check operator pods
kubectl get pods -n tailscale

# Check operator logs
kubectl logs -n tailscale deployment/tailscale-operator

# Test connectivity from a worker pod
kubectl exec -it deployment/stackeye-worker -n stackeye -- \
  nc -zv a1-ops-prd.tailnet-xxxx.ts.net 5432
```

## Troubleshooting

### Operator not starting
- Verify OAuth credentials are correct
- Check that tags exist in ACL policy
- Review operator logs: `kubectl logs -n tailscale deployment/tailscale-operator`

### Cannot connect to database
- Verify ACL rules allow traffic from `tag:stackeye-worker` to database
- Check that database is tagged correctly in Tailscale
- Ensure MagicDNS is enabled in Tailscale admin

### Nodes not appearing in admin
- Verify OAuth client has correct scopes
- Check that `tag:k8s-operator` owns `tag:k8s`
- Review operator logs for authentication errors

## Related Tasks

- Task #5977: Install Tailscale on stackeye-nyc3
- Task #5978: Install Tailscale on stackeye-sfo3
- Task #5979: Configure Tailscale ACLs

## References

- [Tailscale Kubernetes Operator](https://tailscale.com/kb/1236/kubernetes-operator)
- [Tailscale ACL Tags](https://tailscale.com/kb/1068/acl-tags)
- [Tailscale OAuth Clients](https://tailscale.com/kb/1215/oauth-clients)
