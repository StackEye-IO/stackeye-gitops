# Tailscale for Worker Clusters

This directory contains Tailscale configuration for enabling worker pods on NYC3 and SFO3 clusters to access services on the Tailscale network (PostgreSQL and Valkey on a1-ops-prd).

## Connectivity Approach: Tailscale Sidecar

**Recommended**: Each worker pod runs a Tailscale sidecar container that joins the Tailscale network directly. This provides:
- Direct Tailscale connectivity per pod
- MagicDNS resolution for `stackeye-db-*` and `stackeye-valkey-*` hostnames
- No complex cluster networking configuration
- Works with userspace networking (no NET_ADMIN required)

## Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Tailscale Network                                │
│                                                                             │
│  ┌──────────────────────┐    ┌──────────────────────┐    ┌────────────────┐│
│  │    a1-ops-prd        │    │   stackeye-nyc3      │    │ stackeye-sfo3  ││
│  │  (Services Origin)   │    │   (Worker Cluster)   │    │(Worker Cluster)││
│  │                      │    │                      │    │                ││
│  │ ┌──────────────────┐ │    │ ┌──────────────────┐ │    │ ┌────────────┐ ││
│  │ │ stackeye-db-*    │ │    │ │  Worker Pod      │ │    │ │ Worker Pod │ ││
│  │ │ stackeye-valkey-*│◄┼────┼─┤  ┌────────────┐  │ │    │ │ ┌────────┐ │ ││
│  │ └──────────────────┘ │    │ │  │ Tailscale  │  │ │    │ │ │Tailscal│ │ ││
│  │                      │    │ │  │ Sidecar    │  │ │    │ │ │Sidecar │ │ ││
│  │ (Tailscale Operator  │    │ │  └────────────┘  │ │    │ │ └────────┘ │ ││
│  │  exposes services)   │    │ └──────────────────┘ │    │ └────────────┘ ││
│  └──────────────────────┘    └──────────────────────┘    └────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

## Directory Contents

| File | Description |
|------|-------------|
| `secrets-template.yaml` | Tailscale auth key secret template |
| `nyc3/connector.yaml` | Tailscale Connector for NYC3 (infrastructure) |
| `sfo3/connector.yaml` | Tailscale Connector for SFO3 (infrastructure) |

## Prerequisites

1. **Tailscale Operator** installed on each cluster
2. **MagicDNS** enabled in Tailscale admin (https://login.tailscale.com/admin/settings/dns)
3. **ACL Policy** configured to allow access:

```json
{
  "acls": [
    {
      "action": "accept",
      "src": ["tag:stackeye-worker"],
      "dst": ["tag:stackeye-db:5432", "tag:stackeye-cache:6379"]
    }
  ]
}
```

## Step 1: Create Tailscale Auth Key Secret

The Tailscale sidecar needs an auth key to join the Tailscale network.

### Generate Auth Key

1. Go to https://login.tailscale.com/admin/settings/keys
2. Create a new auth key with these settings:
   - **Reusable**: Yes (for multiple pod replicas)
   - **Ephemeral**: Yes (auto-expire when pods terminate)
   - **Tags**: `tag:stackeye-worker`
   - **Expiration**: 90 days (rotate before expiry)

### Create the Secret

```bash
# Edit secrets-template.yaml and replace <TAILSCALE_AUTH_KEY> with your key

# Apply to NYC3 cluster (all namespaces)
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl apply -f secrets-template.yaml

# Apply to SFO3 cluster (all namespaces)
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl apply -f secrets-template.yaml
```

## Step 2: Sidecar Configuration (via Helm Values)

The Tailscale sidecar is configured in the environment values files (`environments/<env>/values.yaml`):

```yaml
worker:
  tailscale:
    enabled: true
    image:
      repository: tailscale/tailscale
      tag: latest
    authSecretName: tailscale-auth
    authSecretKey: authkey
    userspace: true          # No NET_ADMIN required
    acceptRoutes: true       # Accept routes from other Tailscale nodes
    resources:
      requests:
        cpu: 10m
        memory: 64Mi
      limits:
        cpu: 100m
        memory: 128Mi
```

ArgoCD will automatically deploy the sidecar when it syncs the worker applications.

## Step 3: Verify Connectivity

After the secrets are created and pods are deployed:

```bash
# Check pod has tailscale sidecar
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl get pods -n stackeye -l app=stackeye-worker

# Exec into worker pod and test database connectivity
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl exec -n stackeye \
  deploy/stackeye-worker -c tailscale -- tailscale status

# Test database connection from worker container
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl exec -n stackeye \
  deploy/stackeye-worker -c worker -- \
  sh -c 'nc -zv stackeye-db-prd 5432'
```

---

## Tailscale Connectors (Infrastructure)

The connectors are deployed for cluster-level Tailscale infrastructure but are **not required** for worker connectivity when using sidecars.

### Connector Deployment

#### NYC3 Cluster

```bash
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl apply -f nyc3/connector.yaml
```

#### SFO3 Cluster

```bash
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl apply -f sfo3/connector.yaml
```

### Connector Verification

```bash
# NYC3
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl get connectors -n tailscale

# SFO3
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl get connectors -n tailscale
```

After deployment, connectors appear at https://login.tailscale.com/admin/machines:
- `stackeye-nyc3-egress` (100.112.85.33)
- `stackeye-sfo3-egress` (100.76.103.107)

### Connector Configuration

| Cluster | Hostname | Tags | Subnet Routes |
|---------|----------|------|---------------|
| NYC3 | `stackeye-nyc3-egress` | `tag:stackeye-worker` | `100.64.0.0/10` (Tailscale CGNAT) |
| SFO3 | `stackeye-sfo3-egress` | `tag:stackeye-worker` | `100.64.0.0/10` (Tailscale CGNAT) |

## Services Accessible via Connectors

| Service | Tailscale Hostname | Port |
|---------|-------------------|------|
| PostgreSQL (dev) | `stackeye-db-dev` | 5432 |
| PostgreSQL (stg) | `stackeye-db-stg` | 5432 |
| PostgreSQL (prd) | `stackeye-db-prd` | 5432 |
| Valkey (dev) | `stackeye-valkey-dev` | 6379 |
| Valkey (stg) | `stackeye-valkey-stg` | 6379 |
| Valkey (prd) | `stackeye-valkey-prd` | 6379 |

## Route Approval (Required)

After deploying the connectors, the subnet routes (`100.64.0.0/10`) must be approved in the Tailscale admin console:

1. Go to https://login.tailscale.com/admin/machines
2. Find `stackeye-nyc3-egress` and `stackeye-sfo3-egress`
3. Click on each machine
4. In the "Subnets" section, approve the `100.64.0.0/10` route
5. Alternatively, add auto-approval to your ACL policy:

```json
{
  "autoApprovers": {
    "routes": {
      "100.64.0.0/10": ["tag:stackeye-worker"]
    }
  }
}
```

### Verify Route Approval

Test that the connector can reach the database:

```bash
# From the connector pod itself
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl exec -n tailscale \
  $(kubectl get pods -n tailscale -l app.kubernetes.io/name=connector -o name --kubeconfig=~/.kube/mattox/stackeye-nyc3) \
  -c tailscale -- tailscale ping stackeye-db-prd
```

Expected output: `pong from stackeye-db-prd (100.x.x.x) via DERP(ord) in XXms`

## Troubleshooting

### Connector Not Appearing in Tailscale Admin

1. Check operator logs:
   ```bash
   kubectl logs -n tailscale deployment/operator
   ```
2. Verify OAuth credentials are valid
3. Check that `tag:k8s-operator` can assign `tag:stackeye-worker`

### DNS Resolution Fails

1. Ensure MagicDNS is enabled in Tailscale admin
2. Verify connector is connected (check Tailscale admin machines list)
3. Test from a pod in the cluster

### Connection Timeout

1. Verify ACL allows `tag:stackeye-worker` to access the service
2. Check that the target service is tagged correctly
3. Review Tailscale operator logs for connection errors

## Related Documentation

- [Tailscale Connectivity Reference](../../docs/tailscale-connectivity.md)
- [Tailscale Setup Guide](../../docs/tailscale-setup.md)
- [Worker Secrets Template](../worker-secrets-template.yaml)
