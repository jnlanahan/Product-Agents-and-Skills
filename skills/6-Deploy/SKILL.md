---
name: 6-Deploy
when_to_use: "User says 'ship to prod', 'go live', 'deploy this', 'deploy to production', or types /6-Deploy."
disable-model-invocation: true
description: MUST BE USED when the user wants to deploy a project to production for the first time, or onboard an existing app's deploy story. Covers pre-flight checks, account setup, env vars, custom domain + SSL, third-party reconfigurations (webhooks, allowed origins, email DNS), post-deploy smoke tests, and runbook generation. Heavy on step-by-step browser guidance. Trigger on `/6-Deploy`, "ship to prod", "go live", "deploy this".
---

# /6-Deploy

You walk the user — assumed to be a developer with limited deployment experience — through a full production deploy end-to-end. The two supported platforms are **Vercel** and **Railway** — co-equal options with no default. If neither is already configured, ask the user which one they want before doing anything else. Every external step (account creation, dashboard navigation, DNS records) gets explicit numbered instructions like "1. Open https://vercel.com (or https://railway.com). 2. Click 'Add New Project'." The user should be able to follow this skill without prior deploy experience.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- The pre-flight checklist (Phase 0) must fully pass before any deploy command runs — do not skip or defer any item.
- Confirm the target environment (staging vs. production) explicitly with the user before executing deploy commands.
- If any post-deploy health check fails, stop immediately and run `/6-Deploy-Rollback` — do not attempt hotfixes on a broken production deploy.

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
- [ ] `/5-Validate-Production-Readiness` has been run AND Critical findings are addressed
- [ ] `/4-Build-Monitoring` has been run (Sentry + PostHog wired)
- [ ] A health-check endpoint exists at `/api/health` (or you'll add one in Phase 2)

### Phase 1: Choose & set up the deploy platform

Determine the platform:

| Detected | Action |
|---|---|
| Nothing | **Ask the user: Vercel or Railway?** No default — present both as co-equal (Vercel = serverless, Next.js-native; Railway = long-running container, good for workers/websockets/non-Next.js). Wait for their choice before proceeding. |
| Vercel/Railway/Render/Fly detected | Use the existing platform |
| Multiple deploy configs (e.g., `vercel.json` + `railway.json`) | Stop, ask which is real, archive the other |

Then walk the user through account setup. Use the relevant section below.

→ See [platform-setup-steps.md](references/platform-setup-steps.md) for verbatim browser instructions for **both** Vercel and Railway account setup and env var wiring. Use the section matching the user's chosen platform.

Use the relevant section from the reference file and deliver it verbatim to the user. Wait for confirmation before proceeding.

### Phase 2: Wire env vars + add health check

Claude-side: open `.env.example` and list every required env var the user needs to provide.

→ See [platform-setup-steps.md](references/platform-setup-steps.md) "Env Vars" section — it has the variable table (where to find each value) plus platform-specific steps for **Vercel** and **Railway**. Deliver the table and the steps for the user's platform, then wait for "done".

Claude-side: if `/api/health` doesn't exist, add it now (use the project's framework idiom — Next.js `app/api/health/route.ts`). Commit and push. Both Vercel and Railway auto-deploy on every push to main once the GitHub repo is connected.

### Phase 3: First deploy + verify it's running

Claude-side: confirm the build succeeded (ask the user to check the platform's "Deployments" tab — Vercel's Deployments tab or Railway's service Deployments — and report any errors).
- Build error → look at the log, fix locally, push again
- Runtime error → check env vars are correct, check Sentry for stack trace

Then:

> **Verify the deploy is live:**
>
> 1. Find your live URL:
>    - **Vercel** → click the latest deployment → you'll see a `*.vercel.app` URL
>    - **Railway** → service → Settings → Networking → "Generate Domain" if you haven't → you'll see a `*.up.railway.app` URL
> 2. Visit `https://<that-url>/api/health` — you should see `{"status":"ok",...}`
> 3. Visit `https://<that-url>` — you should see your homepage
> 4. Reply with the URL and "live" if it's working

### Phase 4: Wire third-party services for production

This is the most-forgotten phase. The user has services configured for `localhost`; production needs them updated. Walk through each service the project uses (skip ones not detected).

→ See [platform-setup-steps.md](references/platform-setup-steps.md) "Third-Party Service Wiring for Production" section for verbatim steps covering Better Auth (Google OAuth) authorized domains, Stripe webhook endpoint, Resend domain verification, and Sentry alert setup. PostHog requires no extra setup — it auto-detects environment from the host.

### Phase 5: Custom domain (optional but recommended for launch)

If the user wants a custom domain (e.g., `myapp.com` instead of `myapp.vercel.app` or `myapp.up.railway.app`):

→ See [platform-setup-steps.md](references/platform-setup-steps.md) "Custom Domain Setup" section for verbatim CNAME and DNS steps for both Vercel and Railway.

Then **redo the Phase 4 third-party updates with the new domain** — Google OAuth redirect URIs, Stripe webhook URL, Resend sender address all need the custom domain added.

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
- [ ] `BETTER_AUTH_URL` is set to the production URL (not localhost)
- [ ] Google OAuth redirect URI includes the production domain
- [ ] Resend domain status is green

#### C. End-to-end smoke test
- [ ] **Sign up** with a brand-new account on production → check your Neon DB `user` table for the new record
- [ ] **Sign in** with same credentials (Google Sign-In on production)
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
- [ ] CI runs on every PR (typecheck + tests). If not, add `.github/workflows/test.yml` — see [platform-setup-steps.md](references/platform-setup-steps.md) "CI Workflow" for the exact YAML.
- [ ] Production logs accessible (Vercel → project → Functions tab / Runtime Logs; Railway → service → Deployments → View Logs / Observability tab)
- [ ] **Uptime monitor** configured. Walk the user through BetterStack — see [platform-setup-steps.md](references/platform-setup-steps.md) "Uptime Monitor Setup" for verbatim steps.
- [ ] **Rollback procedure tested**: Vercel → Deployments → previous successful deploy → "..." menu → "Promote to Production"; Railway → service → Deployments → previous deploy → "..." → "Redeploy" (or "Rollback") → confirm rollback works on a non-critical change

#### F. Legal & content
- [ ] Privacy Policy page exists and is linked from footer
- [ ] Terms of Service page exists and is linked from footer
- [ ] If processing EU/UK users: GDPR consent banner OR session-only PostHog
- [ ] Marketing emails: unsubscribe link + `List-Unsubscribe` header in every email

### Phase 7: Generate the runbook

→ See [runbook-template.md](references/6-Deploy-Runbook-template.md) for the full `LAUNCH_RUNBOOK.md` template. Create it at the project root, filling in the production URL, BetterStack status URL, and on-call contacts.

### Phase 8: Sign-off

Show a summary table: each Phase-6 check, status (PASS / FAIL / N/A). If any FAIL, refuse to certify launch — list what's left.

If all PASS:

> ✅ **Production deploy verified. You're ready to launch.**
>
> - Production URL: https://<your-domain>
> - Runbook: `LAUNCH_RUNBOOK.md` (committed to git)
> - Re-run `/0-Next` after launch to track post-launch tasks

## Rules

- **Step-by-step external instructions, always.** Every action outside the codebase gets numbered steps with exact button labels and URLs. Don't say "configure your DNS" — say "open your DNS provider, find DNS settings for yourdomain.com, add a CNAME record with name 'www' and value 'cname.vercel-dns.com' (Vercel) or the CNAME target Railway shows you, save." Use the CNAME target for the user's chosen platform.
- **Pause for confirmation between phases.** Don't dump 50 instructions and walk away. After each user-facing phase, wait for "done" before proceeding.
- **Never hand-wave third-party setup.** Account creation, dashboard navigation, DNS records, env var lookup — all explicit.
- **Pre-flight is non-negotiable.** Don't deploy a build that fails locally.
- **Secrets via platform UI/CLI, never committed.** Even production env files don't go in git.
- **Refuse to certify if Critical issues remain.** This is the last gate. Don't pass things along with "you should fix this later."
- **The runbook lives in the repo, not in Notion.** It rots if it's separate from the code.

## If Something Goes Wrong

- **Build fails during deploy** — read the full build log for the first error; common causes are missing env vars, incompatible Node version, or a missing dependency. Fix locally first and verify the build passes before re-deploying.
- **Custom domain not resolving** — DNS propagation can take up to 48 hours; use `dig yourdomain.com` to check current records; confirm the CNAME or A record matches the platform's instructions exactly.
- **Health check fails post-deploy** — check application logs in the platform dashboard immediately; a failing health check usually means a missing env var, a migration that did not run, or a startup crash. Run `/6-Deploy-Rollback` if the service is degraded for users.
- **Third-party webhooks stop working** — update webhook URLs in Stripe, Resend, and other providers to the new production URL; test each webhook with the provider's test event feature.