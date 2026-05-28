---
name: skills
description: Use to discover what skills are available and which ones make sense right now. Lists all skills grouped by PDLC phase, calls project-state-detector to highlight on-mode skills, and shows one-line descriptions. Read-only.
when_to_use: "User says 'what skills are available', 'what can I run', 'show me all commands', 'what slash commands exist', 'what can I do'."
---

# /skills

Discover what's available without memorizing slash commands.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present.
- Call `project-state-detector` to determine current mode.

## Post-flight

This skill is read-only. Do not append to `progress.md`.

## Procedure

### Step 1: Detect project state

From `project-state-detector`, get:
- Current MODE and MATURITY
- RECOMMENDED_NEXT

### Step 2: List all skills by PDLC phase

Read the frontmatter `name` and `description` fields from each `skills/*/SKILL.md`. Group by phase prefix.

Format:

```
CURRENT MODE: <mode> — recommended next: /<skill>

── SETUP & ORIENTATION ────────────────────────────────
  /start      Initialize a new project and capture context
  /resume     Recover session state at the start of a new chat ★ ON-MODE
  /next       State-aware dashboard: where am I, what's next?
  /skills     This command — discover available skills

── DISCOVER (Phase 1) ─────────────────────────────────
  /discover   Surface a real problem before writing a PRD

── DEFINE (Phase 2) ───────────────────────────────────
  /prd        Synthesize a PRD from current context
  /plan       Turn a PRD into vertical implementation slices
  /measure    Define success metrics and telemetry plan
  /refactor   Find or plan a refactor with safe tiny commits
  /glossary   Extract domain terminology into .claude/glossary.md
  /grill-me   Stress-test a plan with relentless questions

── DESIGN (Phase 3) ───────────────────────────────────
  /architect  Make architecture decisions explicit before building ★ ON-MODE
  /prototype  Generate 3 clickable HTML UI variants to compare

── BUILD (Phase 4) ────────────────────────────────────
  /build-feature    Implement a feature in TDD layers
  /code-map         Map an unfamiliar area of the codebase
  /setup-project    Scaffold a new project from scratch
  /setup-database   Wire a DB, add tables, run migrations
  /add-auth         Add or extend authentication
  /add-payment      Add or extend payments (Stripe)
  /add-files        Add file uploads and storage
  /add-monitoring   Wire Sentry and PostHog
  /add-ai           Add Claude / LLM capabilities
  /add-email        Add transactional email
  /setup-ci         Wire GitHub Actions CI
  /setup-tests      Add a test framework and first tests
  /migrate-from-vibe  Move a vibe-coded app to a real stack

── VALIDATE (Phase 5) ─────────────────────────────────
  /check-production   Pre-launch production-readiness audit
  /triage             Investigate and document a bug
  /uat                Run a structured user acceptance test
  /accessibility      WCAG 2.1 AA audit

── DEPLOY (Phase 6) ───────────────────────────────────
  /deploy         Deploy to production end-to-end
  /feature-flag   Wire feature flags for staged rollout
  /rollback       Generate a rollback plan
  /runbook        Generate an operational runbook
  /handoff        Package PRD + plan for external stakeholders

── LEARN (Phase 7) ────────────────────────────────────
  /post-launch-review  Close the learn loop 2–4 weeks after launch
  /postmortem          Blameless postmortem after an incident

★ = highlighted as on-mode for current project state
```

### Step 3: Highlight on-mode and off-mode skills

After the list, add:

```
RECOMMENDED: /<skill> — <one-line rationale from project-state-detector>

Off-pattern skills (callable, but not the current priority):
  /<skill>  /<skill>
```

## Constraints

Never modify any files. Read-only.
