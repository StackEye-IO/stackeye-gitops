# External Services Setup Guide

This guide covers setting up external services for StackEye and creating their sealed secrets.

## Table of Contents

1. [Stripe (Billing)](#stripe-billing)
2. [AWS SES (Email)](#aws-ses-email)
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

## AWS SES (Email)

AWS Simple Email Service handles transactional emails (alerts, invitations, password resets).

### 1. Set Up AWS SES

1. Go to [AWS SES Console](https://console.aws.amazon.com/ses/)
2. Select your preferred region (us-east-1 recommended for best deliverability)

### 2. Verify Domain

1. Go to **Identities** → **Create identity**
2. Select **Domain**
3. Enter: `stackeye.io`
4. Enable **Easy DKIM** (recommended)
5. Add the DNS records shown:
   - 3 CNAME records for DKIM
   - Optional: Custom MAIL FROM subdomain
6. Wait for verification (usually 24-72 hours)

### 3. Request Production Access

**Note**: New AWS accounts start in sandbox mode (can only send to verified emails).

1. Go to **Account dashboard**
2. Click **Request production access**
3. Fill out the form:
   - **Mail type**: Transactional
   - **Website URL**: https://stackeye.io
   - **Use case description**: Uptime monitoring alerts, team invitations, password resets
   - **Expected volume**: Start with 1,000/day, scale as needed
4. Wait for approval (typically 24-48 hours)

### 4. Create IAM User for SES

1. Go to [IAM Console](https://console.aws.amazon.com/iam/)
2. Click **Users** → **Create user**
3. Name: `stackeye-ses-sender`
4. Attach policy: `AmazonSESFullAccess` (or create custom policy below)
5. Create access keys for programmatic access
6. Copy **Access Key ID** and **Secret Access Key**

**Custom IAM Policy (recommended for least-privilege)**:
```json
{
  "Version": "2012-10-17",
  "Statement": [
    {
      "Effect": "Allow",
      "Action": [
        "ses:SendEmail",
        "ses:SendRawEmail"
      ],
      "Resource": "*",
      "Condition": {
        "StringEquals": {
          "ses:FromAddress": "noreply@stackeye.io"
        }
      }
    }
  ]
}
```

### 5. Create Sealed Secret

```bash
# Create plaintext secret
cat > /tmp/ses-secrets.yaml <<EOF
apiVersion: v1
kind: Secret
metadata:
  name: stackeye-ses-secrets
  namespace: stackeye-prd
  labels:
    app.kubernetes.io/name: stackeye
    app.kubernetes.io/component: api
    environment: prd
type: Opaque
stringData:
  AWS_ACCESS_KEY_ID: "AKIA..."
  AWS_SECRET_ACCESS_KEY: "your-secret-key"
  AWS_REGION: "us-east-1"
  EMAIL_FROM_NAME: "StackEye"
  EMAIL_FROM_EMAIL: "noreply@stackeye.io"
EOF

# Seal the secret
kubeseal --format=yaml \
  --cert ~/.claude/secrets/sealed-secrets-cert.pem \
  < /tmp/ses-secrets.yaml > infrastructure/secrets/a1-ops-prd/sealed-ses-secrets-prd.yaml

# Delete plaintext
rm /tmp/ses-secrets.yaml

# Apply
KUBECONFIG=~/.kube/mattox/a1-ops-prd kubectl apply -f infrastructure/secrets/a1-ops-prd/sealed-ses-secrets-prd.yaml
```

### 6. Sending Limits

| Mode | Daily Limit | Rate Limit |
|------|-------------|------------|
| Sandbox | 200 emails | 1/second |
| Production | Request-based | Scales with reputation |

### 7. Monitoring

Set up CloudWatch alarms for:
- Bounce rate (keep < 5%)
- Complaint rate (keep < 0.1%)
- Sending quota usage

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
| AWS SES | [console.aws.amazon.com/ses](https://console.aws.amazon.com/ses) | [docs.aws.amazon.com/ses](https://docs.aws.amazon.com/ses/) |
| Twilio | [console.twilio.com](https://console.twilio.com) | [twilio.com/docs](https://twilio.com/docs) |
| Auth0 | [manage.auth0.com](https://manage.auth0.com) | [auth0.com/docs](https://auth0.com/docs) |
