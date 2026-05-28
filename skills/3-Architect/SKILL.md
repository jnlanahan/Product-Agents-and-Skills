---
name: architect
description: Use when a feature or product involves novel architecture decisions that warrant explicit review before building. Walks through 5 steps (detect stack → data model → integrations → tradeoffs → output) to produce .claude/architecture.md. Step-decomposed — each step ends at a reviewable checkpoint. Optional for trivial features; required for greenfield products and significant infrastructure changes.
when_to_use: "User says 'what's the architecture', 'design the system', 'help me think through the data model', 'how should we architect this', 'technical design'."
---

# /architect

Make architecture decisions explicit and reviewable before building begins. Works in sessions — stop after any step and resume later.

## Pre-flight

- Check for `.claude/architecture.md`. If it exists, this may be an update run — confirm with the user.
- Read `.claude/prd.md` and `.claude/context.md` if they exist.
- Read `.claude/progress.md` (last 5 entries).
- Call `project-state-detector`; if mode is off-pattern, surface a one-line note (do NOT block).

## Post-flight

- `.claude/architecture.md` is produced by step-05.
- Append to `.claude/progress.md`: timestamp, `/architect`, output path, key decisions, suggested next step.
- If `.claude/progress.md` is missing, create it with a header first.

## How it works

Each step is gated — the PM reviews output before proceeding. Steps are in `steps/`.

```
step-01-detect-stack   → Current stack profile
step-02-data-model     → Data model decisions
step-03-integrations   → Integration points and constraints
step-04-tradeoffs      → Key tradeoffs and rejected alternatives
step-05-output         → Assemble .claude/architecture.md
```

## Orchestration logic

Check `.claude/progress.md` for any prior `/architect` entries to determine resume point.

If no prior architect work: start at step-01.

After each step, ask: "Ready to continue to the next step, or do you want to pause here?"

## When to skip

- Simple CRUD feature on existing stack → skip; use `/plan` directly
- Bug fix or small enhancement → skip entirely
- Full greenfield product or novel architecture → run all 5 steps

## Decision logging

At step-04 (tradeoffs), after producing the tradeoff summary, ask:

> "Are any of these decisions load-bearing enough to log in `.claude/decisions.md`? I'll ask for each one."

Log only decisions the PM explicitly confirms as worth recording.
