---
name: rollback
description: MUST BE USED when a production deploy has introduced a regression and the team needs a rollback plan, or proactively before a risky deploy to document the rollback path. Assesses what changed, classifies rollback complexity, and generates a numbered step-by-step rollback runbook covering code, DB migrations, env vars, and traffic. Trigger on `/rollback`, "rollback plan", "how do I revert production", "undo deploy", "production is broken", "need to rollback".
---

# /rollback

You generate and (when the user is ready to execute) walk through a rollback for a broken production deploy.

## Critical

- Confirm that production is actually broken before initiating rollback — misidentified incidents waste rollback capacity and can cause their own outages.
- Classify whether the rollback involves a database migration reversal before starting; irreversible migrations require a different path than code-only rollbacks.
- Communicate status to users and stakeholders before executing rollback steps — silent rollbacks erode trust.

## Procedure

### Step 1: Assess the situation

Ask:
1. What changed in this deploy? (check all that apply: code, DB migration, env vars, infra/DNS config)
2. What is broken? (specific error message, degraded metric, or complete outage?)
3. Deploy platform: Render, Railway, Vercel, Fly.io, or other?
4. Is there a DB migration in this deploy? If yes, is it additive (add column/table) or destructive (drop/rename)?

### Step 2: Classify rollback complexity

| What changed | Complexity | Strategy |
|---|---|---|
| Code only, no DB migration | Simple | Revert deploy to previous version |
| Code + additive migration (added column or table) | Medium | Revert code; leave migration (backward-compatible schema) |
| Code + destructive migration (dropped column, renamed, deleted rows) | Complex | Revert code first, then compensating migration + possible data restore |
| Env var / secret change only | Simple | Restore previous value + redeploy |
| DNS / CDN config | Varies | Platform-specific; may have propagation delay |

### Step 3: Generate the rollback plan

Output a numbered plan based on the complexity above. Include:
- Exact commands or UI steps
- What to verify after each step
- Estimated time
- Who to notify

Save plan to `.claude/rollback-<date>.md`.

### Step 4: Walk through (if user is ready to execute)

Go step by step. Confirm each step completed before proceeding. Rollbacks have ordering dependencies — do not skip or reorder.

### Step 5: After rollback

> Rollback complete. Next:
> - Confirm the regression is resolved (smoke test the affected feature)
> - Notify stakeholders that the incident is resolved
> - Run `/postmortem` to capture what happened, why, and how to prevent it

---

## Rollback Playbooks

### Code-only: Render

```
1. Render dashboard → your service → Deploys tab
2. Find the last known-good deploy
3. Click "..." → "Rollback to this deploy"
4. Wait for deploy to complete (~2 min)
5. Verify: smoke test the broken feature — confirm resolved
```

### Code-only: Railway

```bash
# List recent deployments
railway deployments list

# Roll back to a specific deployment
railway rollback <deployment-id>
```

Or via Railway dashboard: Project → Service → Deployments → click the stable deploy → Rollback.

### Code-only: Vercel

Vercel dashboard → Project → Deployments → find the stable deploy → "..." → "Promote to Production".

### Code-only: git revert (any platform)

```bash
# Find the last stable commit
git log --oneline -20

# Create a revert commit — safe, doesn't rewrite history
git revert <broken-commit-sha> --no-edit

# If reverting a merge commit:
git revert -m 1 <merge-commit-sha> --no-edit

# Push — this triggers a new deploy
git push origin main
```

### Additive migration: leave it

```
Code has been rolled back.
The new column/table exists in the DB but nothing writes to it. This is safe.

No DB action needed now. Schedule cleanup:
- After 1 week of stability, verify column is unused
- Drop the column in a new migration: ALTER TABLE x DROP COLUMN y;
```

### Destructive migration: compensating migration

```
⚠️ This is the hardest case. Order matters:
1. Roll back the code first (see above) — get the app stable
2. Write a NEW compensating migration that reverses the destructive change:
   - Dropped column → re-add it (data will be null; restore from backup if needed)
   - Renamed column → rename back
   - Deleted rows → restore from backup
3. Apply: npm run db:migrate
4. Verify data integrity
5. Notify stakeholders of any data loss

⚠️ Never edit an already-applied migration file. Always write a new migration forward.
```

### Env var rollback

```
1. Platform env vars panel → restore the previous value
2. Render/Railway will auto-redeploy on env change; Vercel may need a manual redeploy
3. Verify the previous behavior is restored
```

---

## Rules

- **Rollback code before DB** — in most cases, rolled-back code works with both old and new schema (additive). Trying to undo a DB migration while broken code runs causes cascading failures.
- **Never edit applied migrations** — write a compensating migration forward instead.
- **Data loss requires a backup restore** — no migration SQL fixes deleted data. Confirm backup availability before starting a destructive rollback.
- **Revert commits, not resets** — use `git revert` (safe, creates new commit) not `git reset --hard` on shared branches.
- **Notify stakeholders immediately** — users should know about an outage; don't try to fix silently for more than 5 minutes.
- **Run `/postmortem` after** — document what happened, why, and how to prevent recurrence.

## If Something Goes Wrong

- **Code rollback succeeds but the app is still broken** — the bug may be in an env var or external service config, not the code; check recent changes to Stripe, email, or auth provider settings.
- **Database migration cannot be reversed** — if the migration is irreversible (e.g., dropped column, renamed table), rollback is a data recovery exercise, not a deploy revert; escalate immediately and stop attempting code rollbacks.
- **Previous version fails to build** — check if the previous version depended on an env var or service that no longer exists; restore the dependency before deploying the previous version.