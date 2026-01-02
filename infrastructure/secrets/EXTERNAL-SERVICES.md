# External Services Setup Guide

This guide covers setting up external services for StackEye and creating their sealed secrets.

## Table of Contents

1. [Stripe (Billing)](#stripe-billing)
2. [Resend (Email)](#resend-email)
3. [Twilio (SMS)](#twilio-sms)
4. [Auth0 (Authentication)](#auth0-authentication)

---

## Stripe (Billing)

Stripe handles subscription billing, payment processing, and usage metering.

### 1. Create Stripe Account

1. Go to [https://dashboard.stripe.com/register](https://dashboard.stripe.com/register)
2. Complete account registration
3. Verify your email address

### 2. Get API Keys

1. Go to [https://dashboard.stripe.com/apikeys](https://dashboard.stripe.com/apikeys)
2. Copy the following:
   - **Publishable key**: `pk_live_...` (or `pk_test_...` for testing)
   - **Secret key**: Click "Reveal live key" → `sk_live_...` (or `sk_test_...`)

### 3. Create Webhook Endpoint

1. Go to [https://dashboard.stripe.com/webhooks](https://dashboard.stripe.com/webhooks)
2. Click "Add endpoint"
3. Enter endpoint URL: `https://api.stackeye.io/v1/webhooks/stripe`
4. Select events to listen for:
   - `customer.subscription.created`
   - `customer.subscription.updated`
   - `customer.subscription.deleted`
   - `invoice.paid`
   - `invoice.payment_failed`
   - `checkout.session.completed`
5. Click "Add endpoint"
6. Copy the **Signing secret**: `whsec_...`

### 4. Create Sealed Secret

```bash
# Create plaintext secret (delete after sealing!)
cat > /tmp/stripe-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: stackeye-stripe-secrets
  namespace: stackeye-prd
  labels:
    app.kubernetes.io/name: stackeye
    app.kubernetes.io/component: api
    environment: prd
type: Opaque
stringData:
  STRIPE_SECRET_KEY: "sk_live_YOUR_SECRET_KEY"
  STRIPE_PUBLISHABLE_KEY: "pk_live_YOUR_PUBLISHABLE_KEY"
  STRIPE_WEBHOOK_SECRET: "whsec_YOUR_WEBHOOK_SECRET"
EOF

# Seal the secret
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/stripe-secrets.yaml > infrastructure/secrets/a1-ops-prd/sealed-stripe-secrets-prd.yaml

# Delete plaintext immediately
rm /tmp/stripe-secrets.yaml

# Apply to cluster
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-stripe-secrets-prd.yaml
```

### 5. Configure Products and Prices

Create products in Stripe for each plan:

| Plan | Monthly Price | Product Name |
|------|---------------|--------------|
| Free | $0 | StackEye Free |
| Starter | $5 | StackEye Starter |
| Pro | $12 | StackEye Pro |
| Team | $29 | StackEye Team |

For each product, create a recurring price with the monthly amount.

---

## Resend (Email)

Resend handles transactional emails (alerts, invitations, password resets).

### 1. Create Resend Account

1. Go to [https://resend.com/signup](https://resend.com/signup)
2. Complete registration

### 2. Verify Domain

1. Go to [https://resend.com/domains](https://resend.com/domains)
2. Click "Add Domain"
3. Enter: `stackeye.io`
4. Add the DNS records shown (SPF, DKIM, DMARC)
5. Wait for verification (usually 5-10 minutes)

### 3. Get API Key

1. Go to [https://resend.com/api-keys](https://resend.com/api-keys)
2. Click "Create API Key"
3. Name: `stackeye-production`
4. Permission: "Sending access"
5. Domain: `stackeye.io`
6. Copy the API key: `re_...`

### 4. Create Sealed Secret

```bash
# Create plaintext secret
cat > /tmp/email-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: stackeye-email-secrets
  namespace: stackeye-prd
  labels:
    app.kubernetes.io/name: stackeye
    app.kubernetes.io/component: api
    environment: prd
type: Opaque
stringData:
  EMAIL_API_KEY: "re_YOUR_API_KEY"
  EMAIL_FROM_NAME: "StackEye"
  EMAIL_FROM_EMAIL: "noreply@stackeye.io"
EOF

# Seal the secret
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/email-secrets.yaml > infrastructure/secrets/a1-ops-prd/sealed-email-secrets-prd.yaml

# Delete plaintext
rm /tmp/email-secrets.yaml

# Apply
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-email-secrets-prd.yaml
```

---

## Twilio (SMS)

Twilio handles SMS notifications for critical alerts.

### 1. Create Twilio Account

1. Go to [https://www.twilio.com/try-twilio](https://www.twilio.com/try-twilio)
2. Complete registration and verify your phone number

### 2. Get Account Credentials

1. Go to [https://console.twilio.com](https://console.twilio.com)
2. On the dashboard, copy:
   - **Account SID**: `AC...`
   - **Auth Token**: Click to reveal

### 3. Get Phone Number

1. Go to [https://console.twilio.com/us1/develop/phone-numbers/manage/incoming](https://console.twilio.com/us1/develop/phone-numbers/manage/incoming)
2. Click "Buy a number" (or use the trial number)
3. Select a number with SMS capability
4. Copy the number in E.164 format: `+15551234567`

### 4. Configure Status Callback (Optional)

For delivery status tracking:
1. When buying/configuring the number, set the SMS webhook to:
   `https://api.stackeye.io/v1/webhooks/twilio/status`

### 5. Create Sealed Secret

```bash
# Create plaintext secret
cat > /tmp/twilio-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: stackeye-twilio-secrets
  namespace: stackeye-prd
  labels:
    app.kubernetes.io/name: stackeye
    app.kubernetes.io/component: api
    environment: prd
type: Opaque
stringData:
  TWILIO_ACCOUNT_SID: "AC_YOUR_ACCOUNT_SID"
  TWILIO_AUTH_TOKEN: "YOUR_AUTH_TOKEN"
  TWILIO_FROM_NUMBER: "+15551234567"
  TWILIO_STATUS_CALLBACK_URL: "https://api.stackeye.io/v1/webhooks/twilio/status"
EOF

# Seal the secret
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/twilio-secrets.yaml > infrastructure/secrets/a1-ops-prd/sealed-twilio-secrets-prd.yaml

# Delete plaintext
rm /tmp/twilio-secrets.yaml

# Apply
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-twilio-secrets-prd.yaml
```

---

## Auth0 (Authentication)

Auth0 provides OAuth2/OIDC authentication. **Note: Auth0 is optional** - StackEye has built-in JWT authentication that works without Auth0.

### When to Use Auth0

- Enterprise SSO requirements (SAML, LDAP)
- Social login (Google, GitHub, etc.)
- Advanced MFA options
- Compliance requirements (SOC2, HIPAA)

### 1. Create Auth0 Tenant

1. Go to [https://auth0.com/signup](https://auth0.com/signup)
2. Create a tenant (e.g., `stackeye-prod`)
3. Select your region

### 2. Create Application

1. Go to Applications → Create Application
2. Name: `StackEye API`
3. Type: "Regular Web Application"
4. Click Create

### 3. Configure Application

1. In the application settings:
   - **Allowed Callback URLs**: `https://app.stackeye.io/callback`
   - **Allowed Logout URLs**: `https://app.stackeye.io`
   - **Allowed Web Origins**: `https://app.stackeye.io`
2. Save changes

### 4. Get Credentials

From the application settings, copy:
- **Domain**: `your-tenant.auth0.com`
- **Client ID**: `...`
- **Client Secret**: `...`

### 5. Create API

1. Go to Applications → APIs → Create API
2. Name: `StackEye API`
3. Identifier (Audience): `https://api.stackeye.io`
4. Signing Algorithm: RS256

### 6. Create Sealed Secret

```bash
# Create plaintext secret
cat > /tmp/auth0-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: stackeye-auth0-secrets
  namespace: stackeye-prd
  labels:
    app.kubernetes.io/name: stackeye
    app.kubernetes.io/component: api
    environment: prd
type: Opaque
stringData:
  AUTH0_ENABLED: "true"
  AUTH0_DOMAIN: "your-tenant.auth0.com"
  AUTH0_AUDIENCE: "https://api.stackeye.io"
  AUTH0_CLIENT_ID: "YOUR_CLIENT_ID"
  AUTH0_CLIENT_SECRET: "YOUR_CLIENT_SECRET"
EOF

# Seal the secret
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/auth0-secrets.yaml > infrastructure/secrets/a1-ops-prd/sealed-auth0-secrets-prd.yaml

# Delete plaintext
rm /tmp/auth0-secrets.yaml

# Apply
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-auth0-secrets-prd.yaml
```

---

## Environment-Specific Secrets

For dev and staging environments, repeat the process with:
- Test/sandbox API keys (Stripe test mode, etc.)
- Different namespaces (`stackeye-dev`, `stackeye-stg`)
- Appropriate naming (`sealed-stripe-secrets-dev.yaml`, etc.)

### Example: Dev Environment Stripe

```bash
cat > /tmp/stripe-secrets-dev.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: stackeye-stripe-secrets
  namespace: stackeye-dev
stringData:
  STRIPE_SECRET_KEY: "sk_test_..."
  STRIPE_PUBLISHABLE_KEY: "pk_test_..."
  STRIPE_WEBHOOK_SECRET: "whsec_..."
EOF

kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/stripe-secrets-dev.yaml > infrastructure/secrets/a1-ops-prd/sealed-stripe-secrets-dev.yaml

rm /tmp/stripe-secrets-dev.yaml
```

---

## Verification

After creating sealed secrets, verify they're working:

```bash
# Check sealed secrets exist
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl get sealedsecrets -A

# Check secrets were created
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl get secrets -n stackeye-prd | grep stackeye

# Verify a specific key exists (without revealing value)
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl get secret stackeye-stripe-secrets -n stackeye-prd -o jsonpath='{.data}' | jq 'keys'
```

---

## Security Checklist

- [ ] Never commit plaintext secrets to git
- [ ] Delete `/tmp/*.yaml` files immediately after sealing
- [ ] Use test/sandbox keys for dev and staging
- [ ] Rotate production keys periodically
- [ ] Store master sealed-secrets key backup securely (`~/.claude/secrets/sealed-secrets-master-key.yaml`)
- [ ] Restrict Stripe webhook to specific IPs if possible
- [ ] Enable Stripe radar for fraud protection
- [ ] Set up Twilio usage alerts to prevent bill shock

---

## Quick Reference

| Service | Dashboard | Docs |
|---------|-----------|------|
| Stripe | [dashboard.stripe.com](https://dashboard.stripe.com) | [stripe.com/docs](https://stripe.com/docs) |
| Resend | [resend.com](https://resend.com) | [resend.com/docs](https://resend.com/docs) |
| Twilio | [console.twilio.com](https://console.twilio.com) | [twilio.com/docs](https://twilio.com/docs) |
| Auth0 | [manage.auth0.com](https://manage.auth0.com) | [auth0.com/docs](https://auth0.com/docs) |
