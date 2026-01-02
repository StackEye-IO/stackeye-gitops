# StackEye Sealed Secrets

This directory contains sealed secrets for StackEye components. Secrets are encrypted using Bitnami Sealed Secrets and can be safely stored in git.

## Directory Structure

```
secrets/
├── a1-ops-prd/                    # Secrets for a1-ops-prd cluster
│   ├── sealed-api-secrets-dev.yaml
│   ├── sealed-api-secrets-stg.yaml
│   └── sealed-api-secrets-prd.yaml
└── README.md
```

## Secrets by Environment

### a1-ops-prd Cluster

| Environment | Secret Name | Keys | Status |
|-------------|-------------|------|--------|
| dev | stackeye-api-secrets | DATABASE_URL, JWT_SECRET, REDIS_URL | Deployed |
| stg | stackeye-api-secrets | DATABASE_URL, JWT_SECRET, REDIS_URL | Deployed |
| prd | stackeye-api-secrets | DATABASE_URL, JWT_SECRET, REDIS_URL | Deployed |

### Secret Contents

Each `stackeye-api-secrets` contains:

| Key | Description | Source |
|-----|-------------|--------|
| DATABASE_URL | PostgreSQL connection string | CNPG `stackeye-db-app` secret |
| JWT_SECRET | 64-char random string for JWT signing | Generated per environment |
| REDIS_URL | Valkey connection string | Valkey operator secret |

## Applying Sealed Secrets

Sealed secrets are automatically applied by ArgoCD, but can also be applied manually:

```bash
# Apply all secrets for a1-ops-prd
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/

# Apply specific environment
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-api-secrets-prd.yaml
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

2. Seal the secret using the a1-ops-prd certificate:
   ```bash
   kubeseal --format=yaml \
     --cert ~/.claude/secrets/sealed-secrets-cert.pem \
     < plaintext-secret.yaml > infrastructure/secrets/a1-ops-prd/sealed-api-secrets-prd.yaml
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
| stackeye-email-secrets | EMAIL_API_KEY | Resend email service |
| stackeye-twilio-secrets | TWILIO_ACCOUNT_SID, TWILIO_AUTH_TOKEN, TWILIO_FROM_NUMBER | Twilio SMS |
| stackeye-auth0-secrets | AUTH0_DOMAIN, AUTH0_AUDIENCE, AUTH0_CLIENT_ID, AUTH0_CLIENT_SECRET | Auth0 (if enabled) |

## Security Notes

1. **Never commit plaintext secrets** - Always seal secrets before committing
2. **Cluster-specific encryption** - Each cluster has its own encryption key. A secret sealed for a1-ops-prd cannot be decrypted by NYC3 or SFO3
3. **Namespace-scoped by default** - Sealed secrets are bound to their target namespace
4. **Key rotation** - Sealed secrets controller rotates keys every 30 days; old keys are retained for decryption

## Related Documentation

- [Sealed Secrets Setup](../sealed-secrets/README.md)
- [Worker Secrets Template](../worker-secrets-template.yaml)
- [CNPG Secrets Template](../cnpg/secrets-template.yaml)
