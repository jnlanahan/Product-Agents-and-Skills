---
name: 6-Deploy
description: MUST BE USED when the user wants to deploy a project to production for the first time, or onboard an existing app's deploy story. Covers pre-flight checks, account setup, env vars, custom domain + SSL, third-party reconfigurations (webhooks, allowed origins, email DNS), post-deploy smoke tests, and runbook generation. Heavy on step-by-step browser guidance. Trigger on `/deploy`, "ship to prod", "go live", "deploy this".
---

# /deploy

You walk the user — assumed to be a developer with limited deployment experience — through a full production deploy end-to-end. Every external step (account creation, dashboard navigation, DNS records) gets explicit numbered instructions like "1. Open https://railway.app/new. 2. Click 'Deploy from GitHub repo'." The user should be able to follow this skill without prior deploy experience.

## When to Use

- First production deploy of a new app
- Onboarding deploy to an existing app that was only running locally
- Re-deploying to a new platform (rare; treat as deliberate migration)
- Re-launching with major changes (new domain, new pricing, new region)

## When NOT to Use

- Routine deploys after CI is set up — those happen automatically on push to main
- Hot-fix deploys — use the platform's UI/CLI directly

## Procedure (the journey)

The user goes through 8 phases. Each phase has Claude-side actions (you do these) and User-side actions (the user does these in their browser/dashboard, with your step-by-step instructions). You guide them and wait for them to confirm each phase is done before moving on.

### Phase 0: Detect & pre-flight (Claude-side)

In parallel:
- `stack-detector` — framework, build commands, existing deploy target
- `codebase-classifier` — wired/vibe-coded affects readiness
- `secret-scanner` — must be clean before deploying

Read `_stack-preferences.md`.

Pre-flight checklist (verify all before continuing). If any fails, stop and address:

- [ ] `npm run build` succeeds locally
- [ ] `npm run check` (typecheck) succeeds locally
- [ ] `npm test` passes (if tests exist)
- [ ] `.env.example` exists and is complete
- [ ] No committed `.env` or service account JSON (`secret-scanner` clean)
- [ ] `/check-production` has been run AND Critical findings are addressed
- [ ] `/add-monitoring` has been run (Sentry + PostHog wired)
- [ ] A health-check endpoint exists at `/api/health` (or you'll add one in Phase 2)

### Phase 1: Choose & set up the deploy platform

Determine the platform:

| Detected | Action |
|---|---|
| Nothing | Default to **Railway** (preference) |
| Railway/Vercel/Render/Fly | Use the existing platform |
| Multiple deploy configs (e.g., `vercel.json` + `railway.json`) | Stop, ask which is real, archive the other |

Then walk the user through account setup. Use the relevant section below.

#### Phase 1A — Railway (default)

Tell the user, verbatim:

> **Railway account setup — do these in your browser:**
>
> 1. Open https://railway.com (or railway.app — they redirect to the same place)
> 2. Click **"Login"** in the top right
> 3. Sign in with GitHub (recommended) — Railway needs GitHub access to deploy your repo
> 4. Once logged in, click **"New Project"**
> 5. Select **"Deploy from GitHub repo"**
> 6. If this is your first time: click **"Configure GitHub App"** and grant Railway access to the repo you want to deploy
> 7. Select your repo from the list
> 8. Railway will start an initial build — let it run, it will probably fail because env vars aren't set yet (that's fine, we'll fix it next)
> 9. Reply "done" when you're at the project page (you'll see "Build", "Deploy", "Variables" tabs)

Wait for the user to confirm.

Then, if the user is using Railway Postgres (not Neon):

> **Add a Postgres database (skip if you're using Neon):**
>
> 1. In your Railway project, click **"+ New"** in the top right
> 2. Click **"Database"** → **"PostgreSQL"**
> 3. Railway provisions a Postgres instance and adds `DATABASE_URL` to your service automatically
> 4. Reply "done" when the database shows up in your project view

If using Neon: skip — the user already has `DATABASE_URL` from the Neon dashboard.

#### Phase 1B — Vercel (if detected)

> **Vercel account setup:**
>
> 1. Open https://vercel.com/signup
> 2. Sign up with GitHub
> 3. Click **"Add New..."** → **"Project"**
> 4. Import your GitHub repo (you may need to install the Vercel GitHub app first)
> 5. On the configuration screen, leave defaults — Vercel auto-detects Next.js
> 6. **Don't deploy yet** — click **"Environment Variables"** to add vars first (next phase)
> 7. Reply "done" when you're at the env vars screen

#### Phase 1C — Render (if detected)

> **Render account setup:**
>
> 1. Open https://render.com/register
> 2. Sign up with GitHub
> 3. Click **"New +"** → **"Web Service"**
> 4. Connect your GitHub repo
> 5. On the configuration screen, set Build Command and Start Command (Claude will give you the right values)
> 6. **Don't deploy yet** — scroll down to env vars first (next phase)
> 7. Reply "done"

### Phase 2: Wire env vars + add health check

Claude-side: open `.env.example` and list every required env var the user needs to provide.

Then tell the user, verbatim, with their actual platform's UI:

> **Set these env vars in Railway:**
>
> 1. In your Railway project, click your service (the one named after your repo)
> 2. Click the **"Variables"** tab
> 3. Click **"+ New Variable"** for each of the following. Get the values from these locations:
>
> | Variable | Where to get it |
> |---|---|
> | `DATABASE_URL` | Auto-set if you used Railway Postgres. If using Neon: copy from Neon dashboard → your project → Connection Details → Connection string |
> | `STRIPE_SECRET_KEY` | Stripe dashboard → Developers → API keys → **"Reveal live key"** (use `sk_live_*` for production, NOT `sk_test_*`) |
> | `STRIPE_WEBHOOK_SECRET` | We'll set this in Phase 4 after you create the production webhook endpoint. Leave blank for now |
> | `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` | Stripe dashboard → Developers → API keys → publishable key (`pk_live_*`) |
> | `FIREBASE_SERVICE_ACCOUNT` | Firebase console → Project Settings → Service Accounts → "Generate new private key" → paste the FULL JSON (with quotes) |
> | `NEXT_PUBLIC_FIREBASE_API_KEY` (and other `NEXT_PUBLIC_FIREBASE_*`) | Firebase console → Project Settings → General → Your apps → SDK setup and configuration |
> | `RESEND_API_KEY` | Resend dashboard → API Keys → "Create API Key" |
> | `NEXT_PUBLIC_POSTHOG_KEY` | PostHog dashboard → Project Settings → Project API Key |
> | `NEXT_PUBLIC_POSTHOG_HOST` | Usually `https://us.i.posthog.com` (or `https://eu.i.posthog.com` if EU) |
> | `SENTRY_DSN` | Sentry dashboard → Settings → Projects → [your project] → Client Keys (DSN) |
> | `NEXT_PUBLIC_SENTRY_DSN` | Same as above (it's safe to expose) |
> | `SENTRY_AUTH_TOKEN` | Sentry → Settings → Account → Auth Tokens → Create token with `project:releases` scope |
>
> 4. After all variables are set, Railway will redeploy automatically
> 5. Reply "done" when all variables are in

Wait for the user.

Claude-side: if `/api/health` doesn't exist, add it now (use the project's framework idiom — Next.js `app/api/health/route.ts` or Express `app.get('/api/health', ...)`). Commit and push. Railway auto-deploys.

### Phase 3: First deploy + verify it's running

Claude-side: confirm the build succeeded (ask the user to check Railway's "Deployments" tab and report any errors).
- Build error → look at the log, fix locally, push again
- Runtime error → check env vars are correct, check Sentry for stack trace

Then:

> **Verify the deploy is live:**
>
> 1. In your Railway project → click your service → **"Settings"** tab → look for **"Public Networking"**
> 2. Click **"Generate Domain"** to get a `*.up.railway.app` URL
> 3. Visit `https://<that-url>/api/health` — you should see `{"status":"ok",...}`
> 4. Visit `https://<that-url>` — you should see your homepage
> 5. Reply with the URL and "live" if it's working

### Phase 4: Wire third-party services for production

This is the most-forgotten phase. The user has services configured for `localhost`; production needs them updated. Walk through each service the project uses (skip ones not detected).

#### Firebase Auth

> **Add your production domain to Firebase Auth:**
>
> 1. Open https://console.firebase.google.com → your project
> 2. **Authentication** (left sidebar) → **Settings** tab → **Authorized domains**
> 3. Click **"Add domain"** → enter your Railway domain (e.g., `your-app.up.railway.app`) and your custom domain if you have one
> 4. Reply "done"

#### Stripe webhook endpoint

> **Register the production webhook in Stripe:**
>
> 1. Open https://dashboard.stripe.com → **Developers** → **Webhooks**
> 2. Make sure you're in **LIVE mode** (toggle in top-right)
> 3. Click **"+ Add endpoint"**
> 4. **Endpoint URL**: `https://<your-domain>/api/webhooks/stripe`
> 5. **Listen to** these events:
>    - `checkout.session.completed`
>    - `customer.subscription.updated`
>    - `customer.subscription.deleted`
>    - `invoice.payment_succeeded`
>    - `invoice.payment_failed`
> 6. Click **"Add endpoint"**
> 7. On the endpoint page, find **"Signing secret"** → click **"Reveal"** → copy the value (starts with `whsec_`)
> 8. Go back to Railway → Variables → set `STRIPE_WEBHOOK_SECRET` to that value
> 9. Reply "done"

#### Resend domain

> **Verify your sending domain in Resend:**
>
> 1. Open https://resend.com/domains
> 2. Click **"Add Domain"**
> 3. Enter the domain you'll send emails from (e.g., `yourdomain.com` — your custom domain, not `*.up.railway.app`)
> 4. Resend shows you DNS records to add (SPF, DKIM, DMARC)
> 5. Add those records in your DNS provider (Phase 5 covers DNS access)
> 6. Wait for verification (usually < 10 min) — Resend will turn the status green
> 7. Reply "done" when verified

#### Sentry alerts

> **Set up Sentry alerts:**
>
> 1. Open https://sentry.io → your project
> 2. **Settings** → **Alerts** → **Create Alert Rule**
> 3. Rule: "When a new issue is seen" → notify your email/Slack
> 4. Reply "done"

#### PostHog

No extra setup needed — PostHog auto-detects environment from the host.

### Phase 5: Custom domain (optional but recommended for launch)

If the user wants a custom domain (e.g., `myapp.com` instead of `myapp.up.railway.app`):

> **Add your custom domain to Railway:**
>
> 1. In Railway → your service → **Settings** → **Public Networking**
> 2. Click **"+ Custom Domain"** → enter your domain (e.g., `myapp.com`)
> 3. Railway shows you a CNAME record (something like `cname.up.railway.app`)
> 4. **At your DNS provider** (where you bought the domain — Namecheap, Cloudflare, GoDaddy, etc.):
>    - Log in to your DNS provider
>    - Find DNS settings for the domain
>    - Add a CNAME record: name = `@` (or your subdomain) → value = the `cname.up.railway.app` value Railway gave you
>    - Save
> 5. Wait 5-30 minutes for DNS to propagate
> 6. Railway auto-provisions an SSL certificate
> 7. Visit `https://yourdomain.com` to verify — should show your app with valid HTTPS
> 8. Reply "done" when working

Then **redo the Phase 4 third-party updates with the new domain** — Firebase authorized domains, Stripe webhook URL, Resend sender address all need the custom domain added.

### Phase 6: Final verification — the launch checklist

This is the gate before announcing. Walk through every box. Any FAIL must be fixed before the user announces / links from social / onboards a paying customer.

#### A. Domain & SSL
- [ ] `https://<domain>` loads without browser warnings
- [ ] HTTP redirects to HTTPS
- [ ] Both `www` and apex resolve (or one redirects to the other)
- [ ] SSL auto-renews (platform handles this — confirm in dashboard)

#### B. Environment variables
- [ ] All vars from `.env.example` are set in production
- [ ] No `STRIPE_SECRET_KEY` starting with `sk_test_` (must be `sk_live_*`)
- [ ] `STRIPE_WEBHOOK_SECRET` is the **production** secret (different from your local `stripe listen` secret)
- [ ] Firebase project ID is the production project (not dev)
- [ ] Resend domain status is green

#### C. End-to-end smoke test
- [ ] **Sign up** with a brand-new account on production → check Firebase console for new user
- [ ] **Sign in** with same credentials
- [ ] Visit a protected page → confirm access
- [ ] Sign out → protected page redirects
- [ ] **Real Stripe charge**: use a real card for the smallest amount your product allows ($0.50 if possible). Confirm:
   - Charge appears in Stripe dashboard
   - Webhook delivered (Stripe → Webhooks → your endpoint → recent deliveries → "Succeeded")
   - User row in your DB updated correctly
   - Refund the charge after verification
- [ ] **Test email**: trigger an email send. Confirm it lands in Gmail inbox (not spam)

#### D. Monitoring is firing
- [ ] **Trigger a test error**: temporarily add `throw new Error('test')` to a route, deploy, hit it, confirm Sentry receives within 1 min, then revert
- [ ] Stack trace shows your `.tsx` source files (source maps uploaded — minified `.js` means they're broken)
- [ ] Visit a few pages → PostHog captures pageviews
- [ ] Sign in on production → user identified in both Sentry and PostHog
- [ ] Sentry alert configured (Phase 4)

#### E. Operations
- [ ] CI runs on every PR (typecheck + tests). If not, add `.github/workflows/test.yml`:

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

- [ ] Production logs accessible (Railway → service → Logs tab)
- [ ] **Uptime monitor** configured. Walk the user through BetterStack:
   > 1. Open https://betterstack.com → sign up free
   > 2. Click **"Create monitor"** → URL = `https://<your-domain>/api/health`
   > 3. Check interval: 1-3 minutes
   > 4. Add your email/SMS for alerts
   > 5. Reply "done"
- [ ] **Rollback procedure tested**: in Railway → Deployments → previous successful deploy → "Redeploy" → confirm rollback works on a non-critical change

#### F. Legal & content
- [ ] Privacy Policy page exists and is linked from footer
- [ ] Terms of Service page exists and is linked from footer
- [ ] If processing EU/UK users: GDPR consent banner OR session-only PostHog
- [ ] Marketing emails: unsubscribe link + `List-Unsubscribe` header in every email

### Phase 7: Generate the runbook

Create `LAUNCH_RUNBOOK.md` in the project root:

```markdown
# Launch Runbook

**Production URL**: https://<your-domain>
**Status page**: https://<betterstack-status-url>
**Last updated**: <date>

## Where to look when things break

| Symptom | First place to check |
|---|---|
| Site is down | Railway dashboard → service status; BetterStack incidents |
| Errors spiking | Sentry → Issues |
| Payments not processing | Stripe dashboard → Webhooks → recent deliveries |
| Emails not sending | Resend dashboard → Logs |
| Auth not working | Firebase console → Authentication |
| Slow responses | Sentry → Performance |
| Database issues | Neon dashboard → Operations log |

## On-call & alerts

- **Primary**: <user name + contact>
- **Sentry alerts go to**: <channel/email>
- **Uptime alerts go to**: <channel/email>

## Common operations

### Roll back a deploy
1. Railway → Deployments tab → previous successful deploy → Redeploy

### Rotate a leaked secret
1. Generate new secret in the relevant dashboard
2. Update env var in Railway → Variables (auto-redeploys)
3. Revoke the old secret in the original dashboard

### Run a migration in production
- Migrations run on deploy automatically (start command includes `db:migrate`)
- Manual: `DATABASE_URL=<prod-url> npm run db:migrate` from local

### View production logs
- Railway → service → Logs tab
- Or Sentry → Issues for error-only filtering

## Known issues
<list, if any>
```

### Phase 8: Sign-off

Show a summary table: each Phase-6 check, status (PASS / FAIL / N/A). If any FAIL, refuse to certify launch — list what's left.

If all PASS:

> ✅ **Production deploy verified. You're ready to launch.**
>
> - Production URL: https://<your-domain>
> - Runbook: `LAUNCH_RUNBOOK.md` (committed to git)
> - Re-run `/next` after launch to track post-launch tasks

## Rules

- **Step-by-step external instructions, always.** Every action outside the codebase gets numbered steps with exact button labels and URLs. Don't say "configure your DNS" — say "open your DNS provider, find DNS settings for yourdomain.com, add a CNAME record with name '@' and value 'cname.up.railway.app', save."
- **Pause for confirmation between phases.** Don't dump 50 instructions and walk away. After each user-facing phase, wait for "done" before proceeding.
- **Never hand-wave third-party setup.** Account creation, dashboard navigation, DNS records, env var lookup — all explicit.
- **Pre-flight is non-negotiable.** Don't deploy a build that fails locally.
- **Secrets via platform UI/CLI, never committed.** Even production env files don't go in git.
- **Refuse to certify if Critical issues remain.** This is the last gate. Don't pass things along with "you should fix this later."
- **The runbook lives in the repo, not in Notion.** It rots if it's separate from the code.
