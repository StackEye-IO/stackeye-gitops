# Tailscale Connectivity Reference

This document provides connection details for StackEye services exposed via Tailscale.

## Service Endpoints

### PostgreSQL (TimescaleDB)

| Environment | Tailscale Hostname | Port | Status |
|-------------|-------------------|------|--------|
| dev | `stackeye-db-dev` | 5432 | Active |
| stg | `stackeye-db-stg` | 5432 | Active |
| prd | `stackeye-db-prd` | 5432 | Active |

### Valkey (Redis-compatible)

| Environment | Tailscale Hostname | Port | Status |
|-------------|-------------------|------|--------|
| dev | `stackeye-valkey-dev` | 6379 | Active |
| stg | `stackeye-valkey-stg` | 6379 | Active |
| prd | `stackeye-valkey-prd` | 6379 | Active |

## Connection Strings

### PostgreSQL

```bash
# Development
postgres://stackeye:<password>@stackeye-db-dev:5432/stackeye?sslmode=require

# Staging
postgres://stackeye:<password>@stackeye-db-stg:5432/stackeye?sslmode=require

# Production
postgres://stackeye:<password>@stackeye-db-prd:5432/stackeye?sslmode=require
```

### Valkey

```bash
# Development
redis://:<password>@stackeye-valkey-dev:6379/0

# Staging
redis://:<password>@stackeye-valkey-stg:6379/0

# Production
redis://:<password>@stackeye-valkey-prd:6379/0
```

## Prerequisites for Connectivity

### MagicDNS Required

Tailscale MagicDNS must be enabled for hostname resolution:
1. Go to https://login.tailscale.com/admin/settings/dns
2. Enable "MagicDNS"
3. Hostnames will resolve to Tailscale IPs (100.x.x.x range)

### Cluster Requirements

| Cluster | Tailscale Operator | Connector/Egress | Worker Access |
|---------|-------------------|------------------|---------------|
| a1-ops-prd | Installed | Services Exposed | Direct access via ClusterIP |
| stackeye-nyc3 | Installed | Required | Needs Connector for egress |
| stackeye-sfo3 | Installed | Required | Needs Connector for egress |

## Worker Pod Connectivity Options

### Option 1: Tailscale Sidecar (Per-Pod)

Add a Tailscale sidecar to worker pods:

```yaml
containers:
  - name: worker
    image: stackeye/worker:latest
    env:
      - name: DATABASE_URL
        value: "postgres://stackeye:<password>@stackeye-db-prd:5432/stackeye?sslmode=require"
  - name: tailscale
    image: tailscale/tailscale:latest
    env:
      - name: TS_AUTHKEY
        valueFrom:
          secretKeyRef:
            name: tailscale-auth
            key: authkey
      - name: TS_USERSPACE
        value: "true"
```

### Option 2: Cluster Egress Connector (Recommended)

Deploy a Tailscale Connector for cluster-wide egress:

```yaml
apiVersion: tailscale.com/v1alpha1
kind: Connector
metadata:
  name: stackeye-egress
  namespace: tailscale
spec:
  hostname: stackeye-nyc3-egress
  exitNode: false
  subnetRouter:
    advertiseRoutes: []
  tags:
    - tag:stackeye-worker
```

This allows all pods in the cluster to access Tailscale services.

## Verification Commands

### From a1-ops-prd (Direct Access)

```bash
# Test PostgreSQL
kubectl run test-pg --rm -it --image=postgres:17-alpine --restart=Never \
  -n stackeye-prd -- pg_isready -h stackeye-db-tailscale -p 5432

# Test Valkey
kubectl run test-valkey --rm -it --image=valkey/valkey:8.0 --restart=Never \
  -n stackeye-prd -- valkey-cli -h stackeye-valkey-tailscale ping
```

### From NYC3/SFO3 (Requires Connector)

```bash
# After Connector deployment
kubectl run test-pg --rm -it --image=postgres:17-alpine --restart=Never \
  -- pg_isready -h stackeye-db-prd -p 5432

# Latency test
kubectl run test-latency --rm -it --image=postgres:17-alpine --restart=Never \
  -- sh -c "time pg_isready -h stackeye-db-prd -p 5432"
```

## Network Architecture

```
┌─────────────────────────────────────────────────────────────────────────────┐
│                            Tailscale Network                                 │
│                                                                              │
│  ┌──────────────────────┐    ┌──────────────────────┐    ┌────────────────┐ │
│  │    a1-ops-prd        │    │   stackeye-nyc3      │    │ stackeye-sfo3  │ │
│  │  (Services Origin)   │    │   (Worker Cluster)   │    │(Worker Cluster)│ │
│  │                      │    │                      │    │                │ │
│  │ ┌──────────────────┐ │    │ ┌──────────────────┐ │    │ ┌────────────┐ │ │
│  │ │ stackeye-db-dev  │ │    │ │  Connector       │ │    │ │ Connector  │ │ │
│  │ │ stackeye-db-stg  │◄┼────┼─┤  (egress to TS)  │ │    │ │ (egress)   │ │ │
│  │ │ stackeye-db-prd  │ │    │ │                  │ │    │ │            │ │ │
│  │ ├──────────────────┤ │    │ └────────┬─────────┘ │    │ └─────┬──────┘ │ │
│  │ │stackeye-valkey-* │ │    │          │           │    │       │        │ │
│  │ └──────────────────┘ │    │ ┌────────▼─────────┐ │    │ ┌─────▼──────┐ │ │
│  │                      │    │ │  Worker Pods     │ │    │ │Worker Pods │ │ │
│  │ (Tailscale Operator  │    │ └──────────────────┘ │    │ └────────────┘ │ │
│  │  exposes services)   │    │                      │    │                │ │
│  └──────────────────────┘    └──────────────────────┘    └────────────────┘ │
└─────────────────────────────────────────────────────────────────────────────┘
```

## Security Notes

1. **TLS Enforcement**: PostgreSQL is configured with `ssl_min_protocol_version = TLSv1.3`
2. **Authentication**: All connections require password authentication
3. **ACL Policy**: Workers must have `tag:stackeye-worker` to access databases
4. **Credentials**: Stored in `.claude/secrets/stackeye-db-connections.env` (not in git)

## Latency Expectations

| Route | Expected Latency | Notes |
|-------|------------------|-------|
| a1-ops-prd → local | < 1ms | Same cluster |
| NYC3 → a1-ops-prd | 30-50ms | East Coast to Central |
| SFO3 → a1-ops-prd | 40-60ms | West Coast to Central |

## Troubleshooting

### DNS Resolution Fails

1. Verify MagicDNS is enabled in Tailscale admin
2. Check that pods are on the Tailscale network
3. For NYC3/SFO3: Verify Connector is deployed

### Connection Timeout

1. Check ACL policy allows `tag:stackeye-worker` to access databases
2. Verify Tailscale operator is running: `kubectl get pods -n tailscale`
3. Check operator logs: `kubectl logs -n tailscale deployment/operator`

### Authentication Error

1. Verify password in connection string
2. Check pg_hba.conf allows Tailscale subnet
3. CNPG default allows all connections with password auth

## Related Documentation

- [Tailscale Setup Guide](tailscale-setup.md)
- [Tailscale ACL Policy](tailscale-acl.json)
- Credentials: `.claude/secrets/stackeye-db-connections.env`

## Current Status

| Component | Status | Notes |
|-----------|--------|-------|
| PostgreSQL Tailscale Services | Active | All 3 environments |
| Valkey Tailscale Services | Active | All 3 environments |
| NYC3 Connector | Deployed | `stackeye-nyc3-egress` (100.112.85.33) |
| SFO3 Connector | Deployed | `stackeye-sfo3-egress` (100.76.103.107) |

**Note:** Subnet routes must be approved in Tailscale admin for pod connectivity.
See `infrastructure/tailscale/README.md` for route approval instructions.
