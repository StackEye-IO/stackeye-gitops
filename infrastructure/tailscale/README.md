# Tailscale Connectors for Worker Clusters

This directory contains Tailscale Connector manifests for enabling worker pods on NYC3 and SFO3 clusters to access services on the Tailscale network.

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
│  │ │ stackeye-db-*    │ │    │ │  Connector       │ │    │ │ Connector  │ ││
│  │ │ stackeye-valkey-*│◄┼────┼─┤  (egress to TS)  │ │    │ │ (egress)   │ ││
│  │ └──────────────────┘ │    │ │                  │ │    │ │            │ ││
│  │                      │    │ └────────┬─────────┘ │    │ └─────┬──────┘ ││
│  │ (Tailscale Operator  │    │          │           │    │       │        ││
│  │  exposes services)   │    │ ┌────────▼─────────┐ │    │ ┌─────▼──────┐ ││
│  │                      │    │ │  Worker Pods     │ │    │ │Worker Pods │ ││
│  └──────────────────────┘    │ └──────────────────┘ │    │ └────────────┘ ││
│                              └──────────────────────┘    └────────────────┘│
└─────────────────────────────────────────────────────────────────────────────┘
```

## Prerequisites

1. **Tailscale Operator** installed on each cluster
2. **MagicDNS** enabled in Tailscale admin (https://login.tailscale.com/admin/settings/dns)
3. **Subnet Routes Approved** in Tailscale admin (see Route Approval section below)
4. **ACL Policy** configured to allow access:

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

## Deployment

### NYC3 Cluster

```bash
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl apply -f nyc3/connector.yaml
```

### SFO3 Cluster

```bash
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl apply -f sfo3/connector.yaml
```

## Verification

### Check Connector Status

```bash
# NYC3
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl get connectors -n tailscale

# SFO3
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl get connectors -n tailscale
```

### Check Tailscale Admin

After deployment, connectors should appear at https://login.tailscale.com/admin/machines:
- `stackeye-nyc3-egress`
- `stackeye-sfo3-egress`

### Test Connectivity

```bash
# From NYC3 - test PostgreSQL
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl run test-pg --rm -it \
  --image=postgres:17-alpine --restart=Never -- \
  pg_isready -h stackeye-db-prd -p 5432

# From SFO3 - test Valkey
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl run test-valkey --rm -it \
  --image=valkey/valkey:8.0 --restart=Never -- \
  valkey-cli -h stackeye-valkey-prd ping
```

## Connector Configuration

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
