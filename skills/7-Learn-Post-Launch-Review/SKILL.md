---
name: 7-Learn-Post-Launch-Review
description: MUST BE USED 2–4 weeks after a production launch or major feature ship to close the learn loop. Reviews actual metrics against the PRD's success criteria, surfaces surprises, runs a Start/Stop/Continue retro, and produces a ranked action item list and next-iteration ideas. Trigger on `/post-launch-review`, "post-launch", "launch retro", "how did it go", "review after launch", "metrics review", "check the launch".
---

# /post-launch-review

You run a structured post-launch review — the learn loop that closes every workflow. You pull metrics, surface surprises, run a retro, and produce a record of what to do next.

## When to Use

- 2–4 weeks after a production launch (W2, W4)
- 1–2 weeks after a major feature ship (W3)
- Any time the team wants a structured retrospective

## Procedure

### Step 1: Gather context

Read:
- `.claude/prd.md` — original success metrics and goals (if it exists)
- `.claude/plan.md` — original planned scope (if it exists)
- `RUNBOOK.md` — current operational state

Ask the user to share (or paste):
1. **PostHog**: key event counts, funnel drop-offs, any session replay observations
2. **Sentry**: error rate before vs. after launch; top new errors since launch
3. **User feedback**: support tickets, DMs, survey responses, App Store reviews
4. **Business metrics**: activation rate, revenue change, retention impact — whatever is tracked

### Step 2: Assess against goals

Compare actual results to the PRD's success metrics. If no PRD existed, ask:
> What did you expect to happen when you shipped, and what actually happened?

Identify:
- What worked better than expected?
- What underperformed?
- What surprised you (positive or negative)?

### Step 3: Run the retro

Ask the user:
> Fill in one or more items for each category:
>
> **Start** — things to do going forward that you didn't do this time
> **Stop** — things that weren't useful and should be dropped
> **Continue** — things that worked well and should be repeated

Synthesize, deduplicate, and clean up the responses.

### Step 4: Extract action items

From the retro and metric review, produce a ranked action item list:
- **P0**: Critical issues (bugs, UX blockers) → fix immediately
- **P1**: Optimization opportunities → prioritize in next sprint
- **P2**: Metrics to add or monitor → observability improvements
- **Backlog**: Next-iteration feature ideas → `/prd` candidates

Each action item needs an owner and a rough timeline.

### Step 5: Save the review

Save to `.claude/post-launch-review-<date>.md`.

Offer:
> Want me to run `/prd` now with the next-iteration ideas from this review?

---

## Post-Launch Review Template

```markdown
# Post-Launch Review — <feature or product name>
Date: <YYYY-MM-DD>
Reviewed by: <name>

## What launched
<one paragraph: what shipped, when, for whom>

## Results vs. goals

| Metric | Goal | Actual | Delta |
|---|---|---|---|
| <metric from PRD> | <target> | <actual> | <+/-N%> |
| Activation rate | | | |
| Error rate (post-launch vs. before) | | | |
| <other KPI> | | | |

## What worked
-
-

## What didn't
-
-

## Surprises
-
-

## Retro

### Start
-

### Stop
-

### Continue
-

## Action items

| Priority | Action | Owner | Timeline |
|---|---|---|---|
| P0 | | | |
| P1 | | | |
| P2 | | | |

## Next-iteration ideas
- <idea> — informed by <observation>
```

---

## Rules

- **Run this even when the launch went well** — good launches have learnings that compound.
- **Use actual numbers**, not vibes. If metrics aren't available, add "instrument X" as a P2 action item.
- **Every action item needs an owner and timeline** — "we should do X" with no owner isn't an action item.
- **Don't skip surprises** — surprises are where the most actionable learnings come from.
- **Connect to the next cycle** — every post-launch review should produce at least one `/prd` candidate or one new `/next-steps` entry.
- **Blameless** — focus on systems, processes, and decisions. Don't name individuals in the "what didn't work" section.
