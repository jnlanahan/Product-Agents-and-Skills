---
name: next
description: Run anytime to see where the project is, what's stale, and which skill to run next. State-aware dashboard powered by project-state-detector. Your default "what should I do?" command. Read-only.
---

# /next

Your default orientation command. Run it anytime you're unsure what to do next, haven't worked on the project in a while, or just want a one-screen summary of where things stand.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries). If it doesn't exist, note that no progress log has been started.
- Read `.claude/context.md` if it exists.
- Call `project-state-detector`.

## When to Use

- **Don't know what to do next?** → `/next`
- **New chat on an existing project?** → `/next` (or `/resume` for a deeper orientation)
- **Just finished a skill and want to know what's next?** → `/next`
- **Onboarding a teammate?** → point them at the output

## When NOT to Use

- For a deep code-level audit → `/check-production`
- For executing a specific change → use the targeted skill directly
- For full session recovery with context replay → `/resume`

## Procedure

### Step 1: Read state

From `project-state-detector` output, capture:
- MODE and MATURITY
- RECOMMENDED_NEXT
- OFF_PATTERN_SKILLS
- SIGNALS

From `progress.md` (last 5 entries), note:
- Most recent skill and timestamp
- Any open / deferred items mentioned

### Step 2: Check artifact freshness

Glob `.claude/`. For each artifact found:
- If not referenced in `progress.md` for 14+ days while project is active → flag as stale

### Step 3: Output one-screen dashboard

Produce a concise dashboard. Must fit on one screen. Format:

```
MODE: <mode> (<maturity>)
SIGNAL: <one-line evidence>

ARTIFACTS
  ✓ .claude/context.md
  ✓ .claude/prd.md
  ⚠ .claude/plan.md  [stale — last touched 2026-04-15]
  ✗ .claude/architecture.md  [missing]

RECOMMENDED NEXT
  /<skill-name> — <one-line rationale>

ALSO AVAILABLE NOW
  /<skill>  /<skill>  /<skill>

OFF-PATTERN (callable but not the priority)
  /<skill>  /<skill>
```

## Post-flight

This skill is read-only. Do not append to `progress.md`.

## Constraints

- Never modify any files.
- Off-pattern skills are surfaced as information only — never blocked.
- If `.claude/` is entirely missing, the dashboard should say "No project state found. Run `/start` to initialize."
