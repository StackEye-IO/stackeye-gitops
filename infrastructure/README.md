# StackEye Infrastructure

This directory contains infrastructure components for StackEye that are managed directly (not via Helm).

## Components

### CNPG (CloudNativePG) PostgreSQL Clusters

TimescaleDB-enabled PostgreSQL clusters for each environment:

| Environment | Namespace | Instances | Storage |
|-------------|-----------|-----------|---------|
| dev | stackeye-dev | 1 | 8Gi |
| staging | stackeye-staging | 2 | 16Gi |
| prod | stackeye | 3 | 32Gi |

## Prerequisites

Before deploying, ensure the following secrets exist in each namespace:

### 1. Harbor Pull Secret

Copy the existing `harbor-supporttools` secret to each namespace:

```bash
# For dev
kubectl get secret harbor-supporttools -n nexmonyx-prd -o yaml | \
  sed 's/namespace: nexmonyx-prd/namespace: stackeye-dev/' | \
  kubectl apply -f -

# For staging
kubectl get secret harbor-supporttools -n nexmonyx-prd -o yaml | \
  sed 's/namespace: nexmonyx-prd/namespace: stackeye-staging/' | \
  kubectl apply -f -

# For prod
kubectl get secret harbor-supporttools -n nexmonyx-prd -o yaml | \
  sed 's/namespace: nexmonyx-prd/namespace: stackeye/' | \
  kubectl apply -f -
```

### 2. Backup Credentials (S3)

Copy the existing `postgres-backup-credentials` secret to each namespace:

```bash
# For dev
kubectl get secret postgres-backup-credentials -n nexmonyx-prd -o yaml | \
  sed 's/namespace: nexmonyx-prd/namespace: stackeye-dev/' | \
  kubectl apply -f -

# For staging
kubectl get secret postgres-backup-credentials -n nexmonyx-prd -o yaml | \
  sed 's/namespace: nexmonyx-prd/namespace: stackeye-staging/' | \
  kubectl apply -f -

# For prod
kubectl get secret postgres-backup-credentials -n nexmonyx-prd -o yaml | \
  sed 's/namespace: nexmonyx-prd/namespace: stackeye/' | \
  kubectl apply -f -
```

### 3. Superuser Secret (Auto-generated)

CNPG will auto-generate a superuser secret if it doesn't exist. However, you can pre-create it:

```bash
# Generate random password
PASSWORD=$(openssl rand -base64 24)

# Create secret for each environment
for ns in stackeye-dev stackeye-staging stackeye; do
  kubectl create secret generic stackeye-db-superuser -n $ns \
    --from-literal=username=postgres \
    --from-literal=password="$PASSWORD"
done
```

## Deployment

### Manual Deployment

```bash
# Create namespaces
kubectl apply -k infrastructure/cnpg/base/

# Deploy dev cluster
kubectl apply -k infrastructure/cnpg/dev/

# Deploy staging cluster
kubectl apply -k infrastructure/cnpg/staging/

# Deploy prod cluster
kubectl apply -k infrastructure/cnpg/prod/
```

### ArgoCD Deployment

Apply the ArgoCD Application:

```bash
kubectl apply -f infrastructure/argocd-apps/
```

## Verification

```bash
# Check cluster status
kubectl get clusters.postgresql.cnpg.io -A | grep stackeye

# Check pods
kubectl get pods -n stackeye-dev -l cnpg.io/cluster=stackeye-db
kubectl get pods -n stackeye-staging -l cnpg.io/cluster=stackeye-db
kubectl get pods -n stackeye -l cnpg.io/cluster=stackeye-db

# Verify TimescaleDB extension
kubectl exec -it stackeye-db-1 -n stackeye-dev -- psql -U postgres -c "CREATE EXTENSION IF NOT EXISTS timescaledb;"
```

## Connection Details

After deployment, connect using:

| Environment | Service | Port |
|-------------|---------|------|
| dev | stackeye-db-rw.stackeye-dev.svc | 5432 |
| staging | stackeye-db-rw.stackeye-staging.svc | 5432 |
| prod | stackeye-db-rw.stackeye.svc | 5432 |

The application credentials are stored in the auto-generated secret `stackeye-db-app` in each namespace.
