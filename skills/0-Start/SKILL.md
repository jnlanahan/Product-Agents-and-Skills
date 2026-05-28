---
name: start
description: Run once at project initialization to capture context, create .claude/ scaffolding, and get oriented. Re-runnable — updates specific sections if .claude/context.md already exists. Never asks which workflow to use; captures context and suggests a starting point.
when_to_use: "User opens a project for the first time, says 'let's get started', 'set up context', 'initialize this project', 'what is this codebase'."
---

# /start

Initialize a new project. Run this once when you open a fresh codebase or a fresh idea. If you've already run it, re-running lets you update specific sections of `.claude/context.md` without overwriting everything.

## Pre-flight

- Check for `.claude/context.md`. If it exists, this is a re-run — offer to update specific sections rather than overwrite.
- Check for `.claude/progress.md`. If it exists, read the last 3 entries so you can acknowledge prior work.

## Procedure

### Step 1: Create .claude/ if missing

If `.claude/` does not exist, create `.claude/context.md` and `.claude/progress.md` as empty files to establish the directory.

### Step 2: Ask orientation questions

If `.claude/context.md` does not exist (first run), ask these 3–5 questions in a single conversational turn:

1. **What are you doing?** (new product / new feature on existing codebase / rearchitecture / bug sprint / exploration / personal tool)
2. **What do you already have?** (just an idea / discovery notes / existing PRD / existing codebase / live product in production)
3. **Team context?** (solo / small team / cross-functional / external contractors)
4. **Primary deliverable expected?** (working code / PRD / prototype / stakeholder handoff doc / all of the above)
5. **Anything essential to know upfront?** (regulated industry, design partner commitment, hard deadline, etc. — optional)

Do not ask for a workflow choice. Do not present a menu. Just gather context.

If `.claude/context.md` already exists (re-run), ask: "Which section would you like to update? (Stack / Conventions / Constraints / Stakeholders / Glossary pointer / Other)"

### Step 3: Write .claude/context.md

Create or update `.claude/context.md` with the following structure. Only populate sections with real answers — skip sections where the user had nothing to say.

```markdown
---
status: draft
last-reviewed: <today's date>
---

## Stack
<chosen technologies and versions, if known>

## Conventions
<naming, file layout, commit style>

## Constraints
<compliance, performance, team constraints, deadlines>

## Stakeholders
<who reviews, who approves, any external partners>

## Glossary pointer
→ See .claude/glossary.md (if it exists)
```

### Step 4: Seed .claude/progress.md

If `.claude/progress.md` does not exist, create it:

```markdown
# Project Progress Log

Append-only. Each entry: timestamp, skill, output, key decisions, next step.

---

## <today's date> <time> — /start completed
- Output: `.claude/context.md`
- Context captured: <brief summary of what the user told you>
- Next likely: <your suggestion based on their answers>
```

If it already exists, append the standard post-flight entry (see Post-flight below).

### Step 5: Suggest a starting point

Based on the user's answers, suggest one skill to run next. Do not present a list. Make one recommendation with a one-sentence rationale.

| User has | Suggest |
|---|---|
| Just an idea | `/discover` — surface the problem before structuring it |
| Discovery notes already | `/prd` — turn your notes into a structured problem definition |
| An existing PRD | `/plan` — decompose the PRD into implementation slices |
| An existing codebase to add to | `/next` — see what state the project is in |
| A vibe-coded prototype | `/migrate-from-vibe` — production-readiness path |

## Post-flight

- Append to `.claude/progress.md`:
  - Timestamp, `/start`, output path `.claude/context.md`
  - Key decisions: what context was captured
  - Suggested next step: the skill you recommended

## Constraints

- Never ask "which workflow do you want." Capture context, suggest one next step.
- Additive only — never overwrite existing `.claude/` files without explicit user confirmation.
- If the user declines all suggestions, that's fine. The goal is orientation, not commitment.
