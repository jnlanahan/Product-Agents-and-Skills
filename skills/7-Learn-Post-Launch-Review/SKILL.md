---
name: 7-Learn-Post-Launch-Review
when_to_use: "User says 'post-launch', 'launch retro', 'how did the launch go', 'review after launch', 'metrics review', or types /7-Learn-Post-Launch-Review. Best used 2–4 weeks after shipping."
description: MUST BE USED 2–4 weeks after a production launch or major feature ship to close the learn loop. Reviews actual metrics against the PRD's success criteria, surfaces surprises, runs a Start/Stop/Continue retro, and produces a ranked action item list and next-iteration ideas. Trigger on `/7-Learn-Post-Launch-Review`, "post-launch", "launch retro", "how did it go", "review after launch", "metrics review", "check the launch".
---

# /7-Learn-Post-Launch-Review

You run a structured post-launch review — the learn loop that closes every workflow. You pull metrics, surface surprises, run a retro, and produce a record of what to do next.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Wait at least 2 weeks post-launch before running this review — earlier data is too noisy to draw reliable conclusions.
- Have actual metric data available (PostHog, Sentry, revenue) before starting; a review built on gut feel misses the point.
- The goal is honest reflection, not celebration — surface what did not work as clearly as what did.

## When to Use

- 2–4 weeks after a production launch (W2, W4)
- 1–2 weeks after a major feature ship (W3)
- Any time the team wants a structured retrospective

## Procedure

### Step 1: Gather context

Read:
- `.claude/2-Define-PRD.md` — original success metrics and goals (if it exists)
- `.claude/2-Define-Plan.md` — original planned scope (if it exists)
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
- **Backlog**: Next-iteration feature ideas → `/2-Define-PRD` candidates

Each action item needs an owner and a rough timeline.

### Step 5: Save the review

Save to `.claude/7-Learn-Post-Launch-Review-<date>.md`.

Offer:
> Want me to run `/2-Define-PRD` now with the next-iteration ideas from this review?

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
- **Connect to the next cycle** — every post-launch review should produce at least one `/2-Define-PRD` candidate or one new `/0-Next-steps` entry.
- **Blameless** — focus on systems, processes, and decisions. Don't name individuals in the "what didn't work" section.

## If Something Goes Wrong

- **No metric data available** — check PostHog event tracking and confirm events were wired before launch; if data is missing, document the gap and schedule a data-wiring fix before the next review.
- **PRD success criteria were not defined** — frame the review around observable user behavior (retention, task completion) rather than skipping it; update the PRD with success criteria for the next cycle.
- **Team disagrees on whether the launch succeeded** — surface the disagreement explicitly in the review doc; use data to adjudicate where possible and document unresolved disagreements as open questions.