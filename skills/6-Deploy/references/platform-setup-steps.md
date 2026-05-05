---
title: "Platform Setup Steps"
skill: "6-Deploy"
---

# Platform Setup Steps

Verbatim browser instructions for each supported deploy platform and third-party service.

## Railway (default)

```
Railway account setup — do these in your browser:

1. Open https://railway.com (or railway.app — they redirect to the same place)
2. Click "Login" in the top right
3. Sign in with GitHub (recommended) — Railway needs GitHub access to deploy your repo
4. Once logged in, click "New Project"
5. Select "Deploy from GitHub repo"
6. If this is your first time: click "Configure GitHub App" and grant Railway access to the repo you want to deploy
7. Select your repo from the list
8. Railway will start an initial build — let it run, it will probably fail because env vars aren't set yet (that's fine, we'll fix it next)
9. Reply "done" when you're at the project page (you'll see "Build", "Deploy", "Variables" tabs)
```

If the user is using Railway Postgres (not Neon):

```
Add a Postgres database (skip if you're using Neon):

1. In your Railway project, click "+ New" in the top right
2. Click "Database" → "PostgreSQL"
3. Railway provisions a Postgres instance and adds DATABASE_URL to your service automatically
4. Reply "done" when the database shows up in your project view
```

## Vercel

```
Vercel account setup:

1. Open https://vercel.com/signup
2. Sign up with GitHub
3. Click "Add New..." → "Project"
4. Import your GitHub repo (you may need to install the Vercel GitHub app first)
5. On the configuration screen, leave defaults — Vercel auto-detects Next.js
6. Don't deploy yet — click "Environment Variables" to add vars first (next phase)
7. Reply "done" when you're at the env vars screen
```

## Render

```
Render account setup:

1. Open https://render.com/register
2. Sign up with GitHub
3. Click "New +" → "Web Service"
4. Connect your GitHub repo
5. On the configuration screen, set Build Command and Start Command (Claude will give you the right values)
6. Don't deploy yet — scroll down to env vars first (next phase)
7. Reply "done"
```

---

## Env Vars — Railway

```
Set these env vars in Railway:

1. In your Railway project, click your service (the one named after your repo)
2. Click the "Variables" tab
3. Click "+ New Variable" for each of the following. Get the values from these locations:

| Variable | Where to get it |
|---|---|
| DATABASE_URL | Auto-set if you used Railway Postgres. If using Neon: copy from Neon dashboard → your project → Connection Details → Connection string |
| STRIPE_SECRET_KEY | Stripe dashboard → Developers → API keys → "Reveal live key" (use sk_live_* for production, NOT sk_test_*) |
| STRIPE_WEBHOOK_SECRET | We'll set this in Phase 4 after you create the production webhook endpoint. Leave blank for now |
| NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY | Stripe dashboard → Developers → API keys → publishable key (pk_live_*) |
| FIREBASE_SERVICE_ACCOUNT | Firebase console → Project Settings → Service Accounts → "Generate new private key" → paste the FULL JSON (with quotes) |
| NEXT_PUBLIC_FIREBASE_API_KEY (and other NEXT_PUBLIC_FIREBASE_*) | Firebase console → Project Settings → General → Your apps → SDK setup and configuration |
| RESEND_API_KEY | Resend dashboard → API Keys → "Create API Key" |
| NEXT_PUBLIC_POSTHOG_KEY | PostHog dashboard → Project Settings → Project API Key |
| NEXT_PUBLIC_POSTHOG_HOST | Usually https://us.i.posthog.com (or https://eu.i.posthog.com if EU) |
| SENTRY_DSN | Sentry dashboard → Settings → Projects → [your project] → Client Keys (DSN) |
| NEXT_PUBLIC_SENTRY_DSN | Same as above (it's safe to expose) |
| SENTRY_AUTH_TOKEN | Sentry → Settings → Account → Auth Tokens → Create token with project:releases scope |

4. After all variables are set, Railway will redeploy automatically
5. Reply "done" when all variables are in
```

---

## Third-Party Service Wiring for Production

### Firebase Auth — add production domain

```
Add your production domain to Firebase Auth:

1. Open https://console.firebase.google.com → your project
2. Authentication (left sidebar) → Settings tab → Authorized domains
3. Click "Add domain" → enter your Railway domain (e.g., your-app.up.railway.app) and your custom domain if you have one
4. Reply "done"
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
8. Go back to Railway → Variables → set STRIPE_WEBHOOK_SECRET to that value
9. Reply "done"
```

### Resend domain verification

```
Verify your sending domain in Resend:

1. Open https://resend.com/domains
2. Click "Add Domain"
3. Enter the domain you'll send emails from (e.g., yourdomain.com — your custom domain, not *.up.railway.app)
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
Add your custom domain to Railway:

1. In Railway → your service → Settings → Public Networking
2. Click "+ Custom Domain" → enter your domain (e.g., myapp.com)
3. Railway shows you a CNAME record (something like cname.up.railway.app)
4. At your DNS provider (where you bought the domain — Namecheap, Cloudflare, GoDaddy, etc.):
   - Log in to your DNS provider
   - Find DNS settings for the domain
   - Add a CNAME record: name = @ (or your subdomain) → value = the cname.up.railway.app value Railway gave you
   - Save
5. Wait 5-30 minutes for DNS to propagate
6. Railway auto-provisions an SSL certificate
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
        with: { node-version: '24', cache: 'npm' }
      - run: npm ci
      - run: npm run check
      - run: npm test
```

---

## Uptime Monitor Setup (BetterStack)

```
1. Open https://betterstack.com → sign up free
2. Click "Create monitor" → URL = https://<your-domain>/api/health
3. Check interval: 1-3 minutes
4. Add your email/SMS for alerts
5. Reply "done"
```
