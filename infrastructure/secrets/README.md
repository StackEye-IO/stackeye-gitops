# StackEye Sealed Secrets

This directory contains sealed secrets for StackEye components. Secrets are encrypted using Bitnami Sealed Secrets and can be safely stored in git.

## Directory Structure

```
secrets/
├── a1-ops-prd/                    # Secrets for a1-ops-prd cluster (API, external services)
│   ├── sealed-api-secrets-{dev,stg,prd}.yaml
│   ├── sealed-stripe-secrets-{dev,stg,prd}.yaml
│   ├── sealed-ses-secrets-{dev,stg,prd}.yaml
│   └── sealed-twilio-secrets-{dev,stg,prd}.yaml
├── stackeye-nyc3/                 # Worker secrets for NYC3 DOKS cluster
│   └── sealed-worker-secrets-{dev,stg,prd}.yaml
├── stackeye-sfo3/                 # Worker secrets for SFO3 DOKS cluster
│   └── sealed-worker-secrets-{dev,stg,prd}.yaml
└── README.md
```

## Secrets by Environment

### a1-ops-prd Cluster (API Server)

| Environment | Secret Name | Keys | Status |
|-------------|-------------|------|--------|
| dev | stackeye-api-secrets | DATABASE_URL, JWT_SECRET, REDIS_URL | Deployed |
| stg | stackeye-api-secrets | DATABASE_URL, JWT_SECRET, REDIS_URL | Deployed |
| prd | stackeye-api-secrets | DATABASE_URL, JWT_SECRET, REDIS_URL | Deployed |
| dev | stackeye-ses-secrets | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, EMAIL_FROM_NAME, EMAIL_FROM_EMAIL | Deployed |
| stg | stackeye-ses-secrets | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, EMAIL_FROM_NAME, EMAIL_FROM_EMAIL | Deployed |
| prd | stackeye-ses-secrets | AWS_ACCESS_KEY_ID, AWS_SECRET_ACCESS_KEY, AWS_REGION, EMAIL_FROM_NAME, EMAIL_FROM_EMAIL | Deployed |
| dev | stackeye-twilio-secrets | TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_FROM_NUMBER | Deployed |
| stg | stackeye-twilio-secrets | TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_FROM_NUMBER | Deployed |
| prd | stackeye-twilio-secrets | TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_FROM_NUMBER | Deployed |

### stackeye-nyc3 Cluster (Workers)

| Environment | Secret Name | Keys | Status |
|-------------|-------------|------|--------|
| dev | stackeye-worker-secrets | DATABASE_URL, REDIS_URL, DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, DB_SSLMODE, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD | Deployed |
| stg | stackeye-worker-secrets | DATABASE_URL, REDIS_URL, DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, DB_SSLMODE, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD | Deployed |
| prd | stackeye-worker-secrets | DATABASE_URL, REDIS_URL, DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, DB_SSLMODE, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD | Deployed |

### stackeye-sfo3 Cluster (Workers)

| Environment | Secret Name | Keys | Status |
|-------------|-------------|------|--------|
| dev | stackeye-worker-secrets | DATABASE_URL, REDIS_URL, DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, DB_SSLMODE, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD | Deployed |
| stg | stackeye-worker-secrets | DATABASE_URL, REDIS_URL, DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, DB_SSLMODE, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD | Deployed |
| prd | stackeye-worker-secrets | DATABASE_URL, REDIS_URL, DB_HOST, DB_PORT, DB_NAME, DB_USER, DB_PASSWORD, DB_SSLMODE, REDIS_HOST, REDIS_PORT, REDIS_PASSWORD | Deployed |

## Secret Contents

### API Secrets (a1-ops-prd)

Each `stackeye-api-secrets` contains:

| Key | Description | Source |
|-----|-------------|--------|
| DATABASE_URL | PostgreSQL connection string | CNPG `stackeye-db-app` secret |
| JWT_SECRET | 64-char random string for JWT signing | Generated per environment |
| REDIS_URL | Valkey connection string | Valkey operator secret |

### Worker Secrets (NYC3 & SFO3)

Each `stackeye-worker-secrets` contains:

| Key | Description | Source |
|-----|-------------|--------|
| DATABASE_URL | Full PostgreSQL connection string | Constructed from CNPG credentials |
| REDIS_URL | Full Valkey connection string | Constructed from Valkey credentials |
| DB_HOST | Tailscale hostname (e.g., `stackeye-db-dev`) | Static per environment |
| DB_PORT | PostgreSQL port (5432) | Static |
| DB_NAME | Database name (stackeye) | Static |
| DB_USER | Database user (stackeye) | Static |
| DB_PASSWORD | Database password | CNPG `stackeye-db-app` secret |
| DB_SSLMODE | SSL mode (require) | Static |
| REDIS_HOST | Tailscale hostname (e.g., `stackeye-valkey-dev`) | Static per environment |
| REDIS_PORT | Redis port (6379) | Static |
| REDIS_PASSWORD | Valkey password | Valkey operator secret |

**Note**: Workers connect via Tailscale hostnames to reach databases on a1-ops-prd cluster.

## Applying Sealed Secrets

Sealed secrets are automatically applied by ArgoCD, but can also be applied manually:

```bash
# Apply all secrets for a1-ops-prd (API server)
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/

# Apply specific environment secret
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-api-secrets-prd.yaml

# Apply all secrets for NYC3 (workers)
KUBECONFIG=~/.kube/mattox/stackeye-nyc3 kubectl apply -f infrastructure/secrets/stackeye-nyc3/

# Apply all secrets for SFO3 (workers)
KUBECONFIG=~/.kube/mattox/stackeye-sfo3 kubectl apply -f infrastructure/secrets/stackeye-sfo3/
```

## Updating Secrets

To update a secret:

1. Create a plaintext secret file (DO NOT commit this):
   ```yaml
   apiVersion: v1
   kind: Secret
   metadata:
     name: stackeye-api-secrets
     namespace: stackeye-prd
   type: Opaque
   stringData:
     DATABASE_URL: "postgres://..."
     JWT_SECRET: "..."
     REDIS_URL: "redis://..."
   ```

2. Seal the secret using the appropriate cluster certificate:
   ```bash
   # For a1-ops-prd cluster
   kubeseal --format=yaml \
     --cert ~/.claude/secrets/sealed-secrets-cert.pem \
     < plaintext-secret.yaml > infrastructure/secrets/a1-ops-prd/sealed-api-secrets-prd.yaml

   # For NYC3 cluster
   kubeseal --format=yaml \
     --cert ~/.claude/secrets/sealed-secrets-cert-nyc3.pem \
     < plaintext-secret.yaml > infrastructure/secrets/stackeye-nyc3/sealed-worker-secrets-prd.yaml

   # For SFO3 cluster
   kubeseal --format=yaml \
     --cert ~/.claude/secrets/sealed-secrets-cert-sfo3.pem \
     < plaintext-secret.yaml > infrastructure/secrets/stackeye-sfo3/sealed-worker-secrets-prd.yaml
   ```

3. Delete the plaintext file:
   ```bash
   rm plaintext-secret.yaml
   ```

4. Apply the sealed secret:
   ```bash
   KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-api-secrets-prd.yaml
   ```

## Future Secrets

The following secrets will be added when the corresponding services are configured:

| Secret | Keys | Service |
|--------|------|---------|
| stackeye-stripe-secrets | STRIPE_SECRET_KEY, STRIPE_WEBHOOK_SECRET, STRIPE_PUBLISHABLE_KEY | Stripe billing |
| stackeye-auth0-secrets | AUTH0_DOMAIN, AUTH0_AUDIENCE, AUTH0_CLIENT_ID, AUTH0_CLIENT_SECRET | Auth0 (if enabled) |

**Already Deployed:**
- `stackeye-ses-secrets` - AWS SES for email notifications
- `stackeye-twilio-secrets` - Twilio for SMS alerts

## Security Notes

1. **Never commit plaintext secrets** - Always seal secrets before committing
2. **Cluster-specific encryption** - Each cluster has its own encryption key. A secret sealed for a1-ops-prd cannot be decrypted by NYC3 or SFO3
3. **Namespace-scoped by default** - Sealed secrets are bound to their target namespace
4. **Key rotation** - Sealed secrets controller rotates keys every 30 days; old keys are retained for decryption

## Related Documentation

- [Sealed Secrets Setup](../sealed-secrets/README.md)
- [Worker Secrets Template](../worker-secrets-template.yaml)
- [CNPG Secrets Template](../cnpg/secrets-template.yaml)
