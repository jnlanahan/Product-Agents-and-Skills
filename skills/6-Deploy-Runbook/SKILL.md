---
name: runbook
description: MUST BE USED after a successful production deploy to generate an operational runbook for on-call handoff. Reads codebase, deploy config, and monitoring setup to produce a reference doc covering health checks, env vars, startup/shutdown, common failure modes and their fixes, alert response, and rollback steps. Trigger on `/runbook`, "generate runbook", "ops runbook", "on-call handoff", "operational documentation", "incident response guide".
---

# /runbook

You generate an operational runbook for the current project — the first-responder reference for anyone keeping this service healthy.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Generate the runbook after a successful deploy, not before — it must reflect the actual deployed configuration, not a plan.
- Every env var listed must be verified as set in the production environment before the runbook is considered complete.
- Include rollback steps even if rollback is "unlikely" — the person reading the runbook is always under pressure.

## Procedure

### Step 1: Gather context

Run in parallel:
- `stack-detector` — full stack, deploy target, monitoring tools
- Read `render.yaml`, `railway.json`, `Dockerfile`, or `.github/workflows/*.yml` if they exist
- Read the health check endpoint code (typically `app/api/health/route.ts` or `routes/health.ts`)
- Read `.env.example` for the full list of required env vars

### Step 2: Ask about operational state

> A few questions to make the runbook accurate:
> 1. Production URL?
> 2. Primary on-call contact (name + Slack/email)?
> 3. Any scheduled jobs or cron tasks? What do they do?
> 4. What are the most common failure modes you've already seen?
> 5. Any external dependencies that, if they go down, take out your app? (e.g., Neon, Firebase, Stripe, Anthropic)

### Step 3: Generate and save

Produce the runbook using the template below. Save to `RUNBOOK.md` at the project root.

---

## Runbook Template

```markdown
# Runbook — <project name>
Generated: <YYYY-MM-DD>
Last reviewed: <YYYY-MM-DD>

---

## Quick reference

| | |
|---|---|
| **Production URL** | https://... |
| **Deploy platform** | Render / Railway / Vercel |
| **Primary on-call** | Name — slack @handle or email |
| **Backup on-call** | Name — slack @handle or email |
| **Monitoring** | [Sentry](https://sentry.io) · [PostHog](https://app.posthog.com) |
| **DB** | [Neon console](https://console.neon.tech) |

---

## Health check

```bash
curl https://<your-app>/api/health
# Expected: {"status":"ok","timestamp":"<ISO date>"}
# HTTP 200
```

If this fails: the service is down. Go to Common Failure Modes.

---

## Starting, stopping, restarting

### Render
- **Deploy**: push to `main` (auto-deploys) or Render dashboard → Manual Deploy
- **Restart**: Render dashboard → your service → "..." → Restart
- **Stop** (emergency): Render dashboard → your service → Suspend service

### Railway
```bash
railway up            # deploy from local
railway rollback <id> # rollback to a previous deployment
```

---

## Required environment variables

All of the following must be set for the service to start. Missing any will cause startup failure.

| Variable | Purpose |
|---|---|
| `DATABASE_URL` | Neon Postgres connection string |
| `FIREBASE_SERVICE_ACCOUNT` | Firebase admin credentials (full JSON, server-only) |
| `NEXT_PUBLIC_FIREBASE_*` | Firebase client config (6 values) |
| `STRIPE_SECRET_KEY` | Stripe secret key |
| `STRIPE_WEBHOOK_SECRET` | Stripe webhook signing secret |
| `ANTHROPIC_API_KEY` | Anthropic Claude API key (if AI features present) |
| `RESEND_API_KEY` | Resend transactional email key (if email present) |
| `NEXT_PUBLIC_POSTHOG_KEY` | PostHog project key |
| `NEXT_PUBLIC_POSTHOG_HOST` | PostHog host URL |
| `SENTRY_DSN` | Sentry DSN for error tracking |
| `LANGCHAIN_API_KEY` | LangSmith API key (if AI tracing present) |
<!-- Add others from .env.example -->

---

## Common failure modes

### DB connection failure
- **Symptoms**: 500 errors on any data-reading endpoint; Sentry shows `NeonDbError: connection refused`
- **Check**: [Neon console](https://console.neon.tech) — is the DB paused? (free tier auto-pauses after 5 min idle)
- **Fix**: Wake the DB from the Neon console. For recurring pauses: upgrade to a paid Neon tier or configure a keep-alive ping.

### Firebase Auth failure
- **Symptoms**: All sign-ins return 401; Sentry shows `Firebase: auth/...` errors
- **Check**: [Firebase console](https://console.firebase.google.com) → Authentication — is the project enabled?
- **Fix**: Verify `FIREBASE_SERVICE_ACCOUNT` env var is valid JSON. Re-generate service account key if it was rotated.

### Stripe webhook failure
- **Symptoms**: Subscriptions not activating after checkout; Stripe dashboard shows webhook delivery failures
- **Check**: Stripe → Developers → Webhooks → check recent event delivery status
- **Fix**: Verify `STRIPE_WEBHOOK_SECRET` matches the signing secret shown in Stripe dashboard for this endpoint.

### AI / Claude API errors
- **Symptoms**: AI features return 500 or 503; Sentry shows `anthropic: 429` (rate limit) or `529` (overload)
- **Check**: [Anthropic status page](https://status.anthropic.com); [Anthropic console](https://console.anthropic.com) for usage
- **Fix**: Add retry with exponential backoff. Reduce concurrent requests. Consider a shorter `max_tokens` limit.

### Email delivery failure
- **Symptoms**: Welcome/reset emails not arriving; Resend dashboard shows bounces or failures
- **Check**: [Resend dashboard](https://resend.com/emails) → Logs
- **Fix**: Verify DNS records (DKIM/SPF) are correct in Resend → Domains. Check `RESEND_API_KEY` is current.

### Memory / CPU spike (Render/Railway)
- **Symptoms**: Service restarts without error; Render Metrics tab shows memory spike before crash
- **Fix**: Identify the culprit (Render logs → look for large data loads). Scale up instance temporarily. File a fix-and-redeploy.

---

## Alert response

| Alert | Threshold | First response |
|---|---|---|
| Sentry: new error spike | >10 new events/min | Check Sentry — identify error type; if deploy-related → `/rollback` |
| p95 API latency > 3s | Monitor in PostHog or Render Metrics | Check Neon slow query log; check for missing DB indexes |
| Health check failing | Uptime monitor alerts | Check service logs; restart if unresponsive |
| Stripe webhook failures | >3 consecutive in Stripe | Check `STRIPE_WEBHOOK_SECRET`; verify endpoint URL is correct |

---

## Scheduled jobs

<!-- Document any cron jobs, background workers, or queue processors here -->
| Job | Schedule | What it does | How to monitor |
|---|---|---|---|
| (none yet) | — | — | — |

---

## Rollback

See full procedure in [`/rollback` skill] or run `/rollback`.

**Quick summary** (code-only rollback):
```
Render: Dashboard → Deploys → "..." → Rollback to previous deploy
Railway: railway rollback <deployment-id>
Git: git revert <sha> --no-edit && git push origin main
```

For DB migration rollback, read the rollback skill first — ordering matters.

---

## External service dashboards

| Service | Dashboard | What to check |
|---|---|---|
| Neon (DB) | https://console.neon.tech | Connection count, query latency, pause status |
| Firebase | https://console.firebase.google.com | Auth error rate |
| Stripe | https://dashboard.stripe.com | Webhook delivery, payment success rate |
| Sentry | https://sentry.io | New error groups, error rate trend |
| PostHog | https://app.posthog.com | Active users, funnel completion |
| Anthropic | https://console.anthropic.com | Rate limit usage, API status |
| Resend | https://resend.com/emails | Delivery rate, bounce rate |
```

---

## Rules

- **Write for a first responder** — the runbook must work for someone who has never touched this project at 3am.
- **Save to `RUNBOOK.md` at the project root** — version-controlled, visible to the whole team.
- **Keep it current** — update when infrastructure changes. A stale runbook is worse than none (false confidence).
- **Link to external dashboards**, don't embed screenshots — URLs are more durable.
- **Never put credentials in the runbook** — the runbook describes where to find credentials, not what they are.

## If Something Goes Wrong

- **Env var listed in runbook is not set in production** — locate the var in the platform's env settings and add it immediately; do not mark the runbook complete until all vars are verified.
- **Health check endpoint returns errors** — trace the error to its source (missing DB connection, startup crash, missing dependency); fix the underlying issue rather than removing the health check.
- **Runbook is generated before the deploy is complete** — discard it and regenerate after the deploy succeeds; a runbook for a planned state is not a runbook.