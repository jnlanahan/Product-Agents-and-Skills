---
name: 1-Discover
description: Use when the user has a vague or unvalidated idea and needs to surface the real problem before writing a PRD. Walks through 5 steps (frame → customer → problem → stakes → bridge) across sessions, tracking state in .claude/discovery-notes.md. Output feeds directly into /2-Define-PRD. Optional — PMs with confirmed problems skip it.
when_to_use: "User has a vague idea and says 'I want to build something but not sure what', 'help me think through this', 'I have an idea I want to explore', 'not sure what to build'."
---

# /1-Discover

Surface the real problem before structuring it. Works in sessions — you can stop after any step and pick up where you left off.

## Pre-flight

- Check for `.claude/discovery-notes.md`. If it exists, read its frontmatter to find which step to resume from.
- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present.
- Call `project-state-detector`; if mode is off-pattern, surface a one-line note (do NOT block).

## Post-flight

- `.claude/discovery-notes.md` is updated after each step (handled within each step file).
- Append to `.claude/progress.md`: timestamp, `/1-Discover`, step completed, key insights, next step.
- If `.claude/progress.md` is missing, create it with a header first.

## How it works

Discovery is step-decomposed because it spans multiple sessions. Each step ends at a natural pause point.

```
step-01-frame     → What space are we in?
step-02-customer  → Who specifically has this problem?
step-03-problem   → What is the problem, precisely?
step-04-stakes    → Why does it matter now?
step-05-bridge    → Produce a problem statement ready for /2-Define-PRD
```

## Orchestration logic

1. Read `.claude/discovery-notes.md` frontmatter (if it exists):
   - `stepsCompleted`: list of finished steps
   - `lastStep`: the most recently completed step name
   - `nextStep`: the step to run now

2. If no discovery file exists: start at `step-01-frame`.

3. Run the appropriate step from `steps/step-0N-<name>.md`.

4. After each step, update `.claude/discovery-notes.md` frontmatter.

5. At step-05 completion, tell the user: "Your discovery is complete. Run `/2-Define-PRD` — it will read `.claude/discovery-notes.md` and skip the front-door interview."

## State file format

`.claude/discovery-notes.md`:

```markdown
---
stepsCompleted: []
lastStep: none
nextStep: frame
---

# Discovery Notes
[Accumulated content from each step appended below]
```

## When to skip

- PM already has a validated problem statement → skip to `/2-Define-PRD`
- PM is extending a known feature → skip to `/2-Define-PRD` or `/2-Define-Plan`
- PM has existing discovery notes in another format → paste them into context and run `/2-Define-PRD` directly

This skill is fully optional. Never block users from going directly to `/2-Define-PRD`.
