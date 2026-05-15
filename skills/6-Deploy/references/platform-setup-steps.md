---
title: "Platform Setup Steps"
skill: "6-Deploy"
---

# Platform Setup Steps

Verbatim browser instructions for each supported deploy platform and third-party service.

## Vercel (default)

```
Vercel account setup — do these in your browser:

1. Open https://vercel.com/signup
2. Sign up with GitHub (required — Vercel deploys directly from your GitHub repo)
3. Click "Add New..." → "Project"
4. Click "Import" next to your GitHub repo (you may need to install the Vercel GitHub app and grant access first)
5. On the Configure Project screen: Vercel auto-detects Next.js — leave all settings as defaults
6. Do NOT click "Deploy" yet — first go to "Environment Variables" (still on this screen) to add your vars
7. Reply "done" when you're at the env vars section
```

---

## Railway (if already in use)

```
Railway account setup — do these in your browser:

1. Open https://railway.com
2. Click "Login" in the top right
3. Sign in with GitHub
4. Once logged in, click "New Project"
5. Select "Deploy from GitHub repo"
6. If this is your first time: click "Configure GitHub App" and grant Railway access to the repo
7. Select your repo from the list
8. Railway will start an initial build — it may fail because env vars aren't set yet (that's fine)
9. Reply "done" when you're at the project page (you'll see "Build", "Deploy", "Variables" tabs)
```

---

## Env Vars — Vercel

```
Set these env vars in Vercel:

1. In your Vercel project configuration screen (or later: Project → Settings → Environment Variables)
2. Add each variable below. Set the Environment to "Production", "Preview", and "Development" as appropriate.
   Use "Production only" for secrets like live Stripe keys.
3. Get the values from these locations:

| Variable | Where to get it |
|---|---|
| DATABASE_URL | Neon dashboard → your project → Connection Details → Connection string (use the pooled connection string for production) |
| BETTER_AUTH_SECRET | Generate with: openssl rand -base64 32 (a long random string — keep it secret) |
| BETTER_AUTH_URL | Your production domain, e.g. https://yourapp.com (or https://yourapp.vercel.app for now) |
| AUTH_GOOGLE_CLIENT_ID | Google Cloud Console → APIs & Services → Credentials → your OAuth 2.0 Client |
| AUTH_GOOGLE_CLIENT_SECRET | Same location as above |
| STRIPE_SECRET_KEY | Stripe dashboard → Developers → API keys → "Reveal live key" (sk_live_* for production) |
| STRIPE_WEBHOOK_SECRET | We'll set this in Phase 4 after you create the production webhook. Leave blank for now |
| NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY | Stripe dashboard → Developers → API keys → publishable key (pk_live_*) |
| AWS_ACCESS_KEY_ID | AWS Console → IAM → Users → your app user → Security credentials → Access keys |
| AWS_SECRET_ACCESS_KEY | Same location — copy the secret when you create the key (only shown once) |
| AWS_REGION | The region you created your S3 bucket in (e.g. us-east-1) |
| AWS_S3_BUCKET | The name of your S3 bucket |
| CLOUDFRONT_DOMAIN | AWS Console → CloudFront → your distribution → Distribution domain name (e.g. d1234abcd.cloudfront.net) |
| CLOUDFRONT_KEY_PAIR_ID | Only if using signed CloudFront URLs: AWS → CloudFront → Key Management → Public Keys |
| CLOUDFRONT_PRIVATE_KEY | Only if using signed CloudFront URLs: the PEM private key content |
| RESEND_API_KEY | Resend dashboard → API Keys → "Create API Key" (if using Resend for email) |
| NEXT_PUBLIC_POSTHOG_KEY | PostHog dashboard → Project Settings → Project API Key |
| NEXT_PUBLIC_POSTHOG_HOST | Usually https://us.i.posthog.com (or https://eu.i.posthog.com for EU) |
| SENTRY_DSN | Sentry dashboard → Settings → Projects → [your project] → Client Keys (DSN) |
| NEXT_PUBLIC_SENTRY_DSN | Same as SENTRY_DSN (safe to expose) |
| SENTRY_AUTH_TOKEN | Sentry → Settings → Account → Auth Tokens → Create token with project:releases scope |

4. After all variables are set, click "Deploy" — Vercel will build and deploy automatically
5. Reply "done" when the deployment finishes
```

---

## Third-Party Service Wiring for Production

### Better Auth / Google Sign-In — add production redirect URI

```
Update your Google OAuth app to accept production sign-ins:

1. Open https://console.cloud.google.com
2. Navigate to APIs & Services → Credentials → your OAuth 2.0 Client ID
3. Under "Authorized redirect URIs", click "+ Add URI"
4. Add: https://yourdomain.com/api/auth/callback/google
   (also add https://yourapp.vercel.app/api/auth/callback/google if you're using the Vercel domain)
5. Click "Save"
6. Also update BETTER_AUTH_URL in your Vercel env vars to: https://yourdomain.com
7. Reply "done"
```

### Stripe webhook endpoint

```
Register the production webhook in Stripe:

1. Open https://dashboard.stripe.com → Developers → Webhooks
2. Make sure you're in LIVE mode (toggle in top-right)
3. Click "+ Add endpoint"
4. Endpoint URL: https://<your-domain>/api/webhooks/stripe
5. Listen to these events:
   - checkout.session.completed
   - customer.subscription.updated
   - customer.subscription.deleted
   - invoice.payment_succeeded
   - invoice.payment_failed
6. Click "Add endpoint"
7. On the endpoint page, find "Signing secret" → click "Reveal" → copy the value (starts with whsec_)
8. Go back to Vercel → your project → Settings → Environment Variables → update STRIPE_WEBHOOK_SECRET to that value
9. Vercel will redeploy automatically
10. Reply "done"
```

### Resend domain verification

```
Verify your sending domain in Resend:

1. Open https://resend.com/domains
2. Click "Add Domain"
3. Enter the domain you'll send emails from (e.g., yourdomain.com — your custom domain, not *.vercel.app)
4. Resend shows you DNS records to add (SPF, DKIM, DMARC)
5. Add those records in your DNS provider (Phase 5 covers DNS access)
6. Wait for verification (usually < 10 min) — Resend will turn the status green
7. Reply "done" when verified
```

### Sentry alerts

```
Set up Sentry alerts:

1. Open https://sentry.io → your project
2. Settings → Alerts → Create Alert Rule
3. Rule: "When a new issue is seen" → notify your email/Slack
4. Reply "done"
```

---

## Custom Domain Setup

```
Add your custom domain to Vercel:

1. In Vercel → your project → Settings → Domains
2. Type your domain name (e.g., myapp.com) → click "Add"
3. Vercel shows you a CNAME record (value: cname.vercel-dns.com) or an A record
4. At your DNS provider (where you bought the domain — Namecheap, Cloudflare, GoDaddy, etc.):
   - Log in to your DNS provider
   - Find DNS settings for the domain
   - Add a CNAME record: name = www → value = cname.vercel-dns.com
   - For the apex domain (@), add an A record → value = 76.76.21.21
   - Save
5. Wait 5-30 minutes for DNS to propagate
6. Vercel auto-provisions an SSL certificate (Let's Encrypt)
7. Visit https://yourdomain.com to verify — should show your app with valid HTTPS
8. Reply "done" when working
```

---

## CI Workflow (if missing)

```yaml
name: CI
on: [push, pull_request]
jobs:
  test:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with: { node-version: '20', cache: 'npm' }
      - run: npm ci
      - run: npm run typecheck
      - run: npm test
```

Note: Vercel auto-deploys on every push to main — no separate CI deploy step needed.

---

## Uptime Monitor Setup (BetterStack)

```
1. Open https://betterstack.com → sign up free
2. Click "Create monitor" → URL = https://<your-domain>/api/health
3. Check interval: 1-3 minutes
4. Add your email/SMS for alerts
5. Reply "done"
```
