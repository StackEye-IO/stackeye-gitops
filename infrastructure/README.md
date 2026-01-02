# StackEye Infrastructure

This directory contains infrastructure components for StackEye that are managed directly (not via Helm).

## Components

### CNPG (CloudNativePG) PostgreSQL Clusters

TimescaleDB-enabled PostgreSQL clusters for each environment:

| Environment | Namespace | Instances | Storage |
|-------------|-----------|-----------|---------|
| dev | stackeye-dev | 1 | 8Gi |
| stg | stackeye-stg | 2 | 16Gi |
| prd | stackeye-prd | 3 | 32Gi |

## Prerequisites

Before deploying, ensure the following secrets exist in each namespace:

### 1. Harbor Pull Secret

The `harbor-supporttools` secret is automatically provisioned by kubetty.

### 2. Backup Credentials (S3)

Create the `postgres-backup-credentials` secret in each namespace:

```bash
# For all environments
for ns in stackeye-dev stackeye-stg stackeye-prd; do
  kubectl create secret generic postgres-backup-credentials -n $ns \
    --from-literal=ACCESS_KEY_ID="<ACCESS_KEY>" \
    --from-literal=SECRET_ACCESS_KEY="<SECRET_KEY>"
done
```

### 3. Superuser Secret (Auto-generated)

CNPG will auto-generate a superuser secret if it doesn't exist. However, you can pre-create it:

```bash
# Generate random password
PASSWORD=$(openssl rand -base64 24)

# Create secret for each environment
for ns in stackeye-dev stackeye-stg stackeye-prd; do
  kubectl create secret generic stackeye-db-superuser -n $ns \
    --from-literal=username=postgres \
    --from-literal=password="$PASSWORD"
done
```

## Deployment

These databases are **managed directly** (not via ArgoCD/Helm).

```bash
# Create namespaces
kubectl apply -k infrastructure/cnpg/base/

# Deploy dev cluster
kubectl apply -k infrastructure/cnpg/dev/

# Deploy stg cluster
kubectl apply -k infrastructure/cnpg/stg/

# Deploy prd cluster
kubectl apply -k infrastructure/cnpg/prd/
```

## Verification

```bash
# Check cluster status
kubectl get clusters.postgresql.cnpg.io -A | grep stackeye

# Check pods
kubectl get pods -n stackeye-dev -l cnpg.io/cluster=stackeye-db
kubectl get pods -n stackeye-stg -l cnpg.io/cluster=stackeye-db
kubectl get pods -n stackeye-prd -l cnpg.io/cluster=stackeye-db

# Verify TimescaleDB extension
kubectl exec -it stackeye-db-1 -n stackeye-dev -- psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS timescaledb;"
```

## Connection Details

After deployment, connect using:

| Environment | Service | Port |
|-------------|---------|------|
| dev | stackeye-db-rw.stackeye-dev.svc | 5432 |
| stg | stackeye-db-rw.stackeye-stg.svc | 5432 |
| prd | stackeye-db-rw.stackeye-prd.svc | 5432 |

The application credentials are stored in the auto-generated secret `stackeye-db-app` in each namespace.
