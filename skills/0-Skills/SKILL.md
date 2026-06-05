---
name: 0-Skills
description: Use to discover what skills are available and which ones make sense right now. Lists all skills grouped by PDLC phase, calls project-state-detector to highlight on-mode skills, and shows one-line descriptions. Read-only.
when_to_use: "User says 'what skills are available', 'what can I run', 'show me all commands', 'what slash commands exist', 'what can I do'."
---

# /0-Skills

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
  /0-Start      Initialize a new project and capture context
  /0-Resume     Recover session state at the start of a new chat ★ ON-MODE
  /0-Next       State-aware dashboard: where am I, what's next?
  /0-Skills     This command — discover available skills

── DISCOVER (Phase 1) ─────────────────────────────────
  /1-Discover   Surface a real problem before writing a PRD

── DEFINE (Phase 2) ───────────────────────────────────
  /2-Define-PRD        Synthesize a PRD from current context
  /2-Define-Plan       Turn a PRD into vertical implementation slices
  /2-Define-Measurement    Define success metrics and telemetry plan
  /2-Define-Refactor   Find or plan a refactor with safe tiny commits
  /2-Define-Glossary   Extract domain terminology into .claude/2-Define-Glossary.md
  /grill-me   Stress-test a plan with relentless questions

── DESIGN (Phase 3) ───────────────────────────────────
  /3-Architect  Make architecture decisions explicit before building ★ ON-MODE
  /3-Design-Prototype  Generate 3 clickable HTML UI variants to compare

── BUILD (Phase 4) ────────────────────────────────────
  /4-Build-Feature    Implement a feature in TDD layers
  /4-Build-Code-Map         Map an unfamiliar area of the codebase
  /0-Setup-Project    Scaffold a new project from scratch
  /4-Build-Database   Wire a DB, add tables, run migrations
  /4-Build-Auth         Add or extend authentication
  /4-Build-Payments      Add or extend payments (Stripe)
  /4-Build-File-Storage        Add file uploads and storage
  /4-Build-Monitoring   Wire Sentry and PostHog
  /4-Build-AI           Add Claude / LLM capabilities
  /4-Build-Email        Add transactional email
  /4-Build-CI         Wire GitHub Actions CI
  /4-Build-Tests      Add a test framework and first tests
  /4-Build-Migrate-From-Vibe  Move a vibe-coded app to a real stack

── VALIDATE (Phase 5) ─────────────────────────────────
  /5-Validate-Production-Readiness   Pre-launch production-readiness audit
  /5-Validate-Triage             Investigate and document a bug
  /5-Validate-UAT                Run a structured user acceptance test
  /5-Validate-Accessibility      WCAG 2.1 AA audit

── DEPLOY (Phase 6) ───────────────────────────────────
  /6-Deploy         Deploy to production end-to-end
  /6-Deploy-Feature-Flag   Wire feature flags for staged rollout
  /6-Deploy-Rollback       Generate a rollback plan
  /6-Deploy-Runbook        Generate an operational runbook
  /6-Handoff        Package PRD + plan for external stakeholders

── LEARN (Phase 7) ────────────────────────────────────
  /7-Learn-Post-Launch-Review  Close the learn loop 2–4 weeks after launch
  /7-Learn-Postmortem          Blameless postmortem after an incident

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
