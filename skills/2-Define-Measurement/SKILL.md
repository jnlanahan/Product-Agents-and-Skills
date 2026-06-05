---
name: 2-Define-Measurement
description: Use to produce a telemetry and measurement plan before building. Defines success metrics, event schema, telemetry destinations, and failure signals. Output is .claude/measurement.md. Pairs with /4-Build-Monitoring for implementation.
when_to_use: "User says 'define metrics', 'measurement plan', 'what should we track', 'telemetry strategy', 'how do we know this worked', 'define KPIs'."
---

# /2-Define-Measurement

Define how you will know if this feature worked — before you build it. Produces a measurement plan that `/4-Build-Monitoring` can implement.

## Pre-flight

- Read `.claude/2-Define-PRD.md` if it exists — success metrics in the PRD are the starting point.
- Read `.claude/architecture.md` if it exists — it informs what events are technically feasible.
- Read `.claude/context.md` if present.
- Read `.claude/progress.md` (last 5 entries).
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block).

## Post-flight

- Append to `.claude/progress.md`: timestamp, `/2-Define-Measurement`, output path `.claude/measurement.md`, key decisions, suggested next step.
- If `.claude/progress.md` is missing, create it with a header first.

## When to Use

- Before implementing a new feature (define what "working" means before writing code)
- When the PRD has `<TBD>` in its success metrics section
- When the team disagrees on what success looks like

## When to Skip

- Feature has no measurable user-facing effect (pure refactor, infra change)
- Metrics already exist and are documented

## Procedure

### Step 1: Pull success criteria from PRD

Read `.claude/2-Define-PRD.md`. Extract or infer the success criteria. If the PRD doesn't have them, ask the user:

> "What does success look like for this feature in 30 days? In 90 days? How would you know it failed?"

### Step 2: Define success metrics

For each success criterion, define a metric:
- **Name**: human-readable (e.g., "Activation rate")
- **Definition**: exact calculation (e.g., "% of users who complete onboarding within 7 days of signup")
- **Target**: numerical goal (e.g., "≥ 40%")
- **Timeframe**: when to check (e.g., "30 days post-launch")
- **Data source**: PostHog, Sentry, DB query, revenue dashboard

### Step 3: Define event schema

List every user action that needs to be tracked:

| Event name | Trigger | Properties | Destination |
|---|---|---|---|
| `<event_name>` | <what action fires it> | `{ key: type, ... }` | PostHog / Sentry / both |

Event naming convention: `noun_verb` (e.g., `user_signed_up`, `feature_activated`, `payment_completed`).

### Step 4: Define telemetry destinations

- **PostHog** — product analytics, session replay, feature flags, funnels
- **Sentry** — error rates, performance, stack traces
- **Database** — counters or aggregates queried by a dashboard
- **Revenue tracking** — Stripe MRR, churn, ARPU

For each metric, confirm which destination provides the data.

### Step 5: Define failure signals

What tells you something is wrong?

| Signal | Threshold | Action |
|---|---|---|
| Error rate | > 1% of requests | Alert on-call, check Sentry |
| Drop-off at step N | > 50% abandonment | Review session replays in PostHog |
| Latency | p99 > 2s | Check performance trace in Sentry |

### Step 6: Write .claude/measurement.md

```markdown
---
status: draft
last-reviewed: <today's date>
---

# Measurement Plan — <feature name>

## Success Metrics

| Metric | Definition | Target | Timeframe | Source |
|---|---|---|---|---|

## Event Schema

| Event | Trigger | Properties | Destination |
|---|---|---|---|

## Failure Signals

| Signal | Threshold | Action |
|---|---|---|

## Implementation Notes

→ Use `/4-Build-Monitoring` to wire Sentry and PostHog.
→ Reference this file when configuring PostHog dashboards and Sentry alerts.
```

## Pairs with

`/4-Build-Monitoring` — implements the monitoring stack that captures these events. Run `/2-Define-Measurement` first to know what to instrument, then `/4-Build-Monitoring` to wire it.
