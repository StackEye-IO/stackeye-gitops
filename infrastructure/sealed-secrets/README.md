# Sealed Secrets for StackEye

This directory contains documentation for the Sealed Secrets controller deployed on a1-ops-prd cluster for GitOps-friendly secret management.

## Overview

[Sealed Secrets](https://github.com/bitnami-labs/sealed-secrets) allows you to encrypt Kubernetes secrets so they can be safely stored in git. Only the controller running in the cluster can decrypt them.

## Current Deployment

| Component | Status | Namespace |
|-----------|--------|-----------|
| sealed-secrets-controller | Running | kube-system |
| Service | sealed-secrets-controller:8080 | kube-system |
| Metrics | sealed-secrets-controller-metrics:8081 | kube-system |

## Quick Start

### 1. Create a regular Kubernetes secret

```bash
kubectl create secret generic my-secret \
  --from-literal=username=admin \
  --from-literal=password=supersecret \
  --dry-run=client -o yaml > my-secret.yaml
```

### 2. Seal the secret

```bash
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < my-secret.yaml > my-sealed-secret.yaml
```

### 3. Apply the sealed secret

```bash
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f my-sealed-secret.yaml
```

The controller will automatically decrypt it and create the actual Secret.

## Important Files

| File | Description |
|------|-------------|
| `~/.claude/secrets/sealed-secrets-master-key.yaml` | Master key backup (CRITICAL - keep secure) |
| `~/.claude/secrets/sealed-secrets-cert.pem` | Public certificate for sealing secrets |

## Sealing Secrets for Specific Namespaces

By default, sealed secrets are bound to a specific namespace. To seal for a specific namespace:

```bash
kubectl create secret generic my-secret \
  --namespace=stackeye-prd \
  --from-literal=password=secret \
  --dry-run=client -o yaml | \
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  > my-sealed-secret.yaml
```

## Sealing Secrets for Any Namespace (Cluster-Wide)

To create a sealed secret that can be deployed to any namespace:

```bash
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  --scope cluster-wide \
  < my-secret.yaml > my-sealed-secret.yaml
```

## Key Management

### Viewing Current Keys

```bash
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key
```

### Backing Up Keys

```bash
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl get secret -n kube-system \
  -l sealedsecrets.bitnami.com/sealed-secrets-key \
  -o yaml > ~/.claude/secrets/sealed-secrets-master-key.yaml
chmod 600 ~/.claude/secrets/sealed-secrets-master-key.yaml
```

### Disaster Recovery

If the cluster is rebuilt or the controller is reinstalled, restore the master key BEFORE applying any sealed secrets:

```bash
# 1. Restore the master key
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f ~/.claude/secrets/sealed-secrets-master-key.yaml

# 2. Restart the controller to pick up the key
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl delete pod -n kube-system \
  -l app.kubernetes.io/name=sealed-secrets

# 3. Now sealed secrets can be decrypted
```

## Fetching a Fresh Public Certificate

If you need to seal secrets from a new machine:

```bash
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > sealed-secrets-cert.pem
```

## Example: Converting Existing Secret Templates

The repository has several secret templates that can be converted to sealed secrets:

```bash
# Example: Convert tailscale-auth secret
# 1. Create the secret with actual values
cat > /tmp/tailscale-auth.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: tailscale-auth
  namespace: stackeye-prd
type: Opaque
stringData:
  authkey: "tskey-auth-xxxxx"
EOF

# 2. Seal it
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/tailscale-auth.yaml > infrastructure/tailscale/sealed-tailscale-auth-prd.yaml

# 3. Clean up plaintext
rm /tmp/tailscale-auth.yaml
```

## Troubleshooting

### Secret not being created

Check the controller logs:

```bash
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl logs -n kube-system \
  -l app.kubernetes.io/name=sealed-secrets
```

### "no key could decrypt secret" error

The sealed secret was created with a different key. Either:
1. Restore the original key from backup
2. Re-seal the secret with the current certificate

### Certificate expired

Fetch a new certificate and re-seal affected secrets:

```bash
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubeseal --fetch-cert \
  --controller-name=sealed-secrets-controller \
  --controller-namespace=kube-system \
  > ~/.claude/secrets/sealed-secrets-cert.pem
```

## Security Notes

1. **Master key is critical**: Without the master key backup, sealed secrets cannot be recovered after a disaster
2. **Public certificate is not sensitive**: Can be shared with developers for sealing secrets
3. **Sealed secrets are namespace-scoped by default**: A sealed secret for `stackeye-prd` cannot be applied to `stackeye-dev`
4. **Rotation**: Keys are automatically rotated every 30 days, but old keys are kept for decrypting existing secrets

## Related Files

- `infrastructure/cnpg/secrets-template.yaml` - Database secrets template
- `infrastructure/worker-secrets-template.yaml` - Worker secrets template
- `infrastructure/tailscale/secrets-template.yaml` - Tailscale auth template
