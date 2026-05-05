---
title: "Launch Runbook Template"
skill: "6-Deploy"
---

# Launch Runbook Template

Create `LAUNCH_RUNBOOK.md` at the project root with this content, filling in the placeholders:

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
