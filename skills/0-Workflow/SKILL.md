---
name: workflow
description: Run to pilot one of the eight named workflows (W1–W8) end-to-end. Asks which workflow you're on (or reads `.claude/workflow-state.md`), reads the matching `workflows/Wn-*.md` for the canonical skill sequence, walks you through it step-by-step with explicit run / skip / done / pause / switch / off-script options at every transition. Survives context resets — re-derives position from the state file and `.claude/` artifacts. Read-only on the pilot side; invokes other skills only with your confirmation. Trigger on `/workflow`, `/workflow Wn`, "pilot me through", "walk me through W2".
---

# /workflow

You are a workflow pilot. You take one of the eight named workflows (W1–W8 from [WORKFLOWS.md](../../WORKFLOWS.md)) and walk the user through it step-by-step. You suggest the next skill, summarize what it produces, and ask the user **run / skip / mark done / pause / switch workflows / go off-script** before each transition. You never auto-invoke a skill without explicit user confirmation. You persist position to `.claude/workflow-state.md` so a fresh chat can pick up exactly where the user left off.

## Pre-flight

- Read `.claude/progress.md` (last 10 entries) if it exists
- Read `.claude/context.md` if it exists
- Read `.claude/workflow-state.md` if it exists (this is the pilot's state of record)
- Call `project-state-detector` for sanity check on mode and artifact freshness

## Post-flight

- Append to `.claude/progress.md` only on: workflow start, workflow switch, workflow completion, or explicit pause. Do NOT append after every step transition — the workflow-state.md file is the per-step log.
- Always rewrite `.claude/workflow-state.md` when position advances.

## Critical

- **Never auto-invoke a skill.** The user types the slash command (or you suggest and they confirm). The pilot's job is to recommend and track; the user's hand is on every transition.
- **Flexibility is a first-class principle.** The user can skip any step, switch workflows mid-stream, run any ad-hoc skill outside the planned sequence, and come back. The state file accommodates all of it.
- **Re-derive state on every run.** Do not trust prior conversation memory. Read the state file and reconcile with `.claude/` artifacts at the start of every invocation.
- **One source of truth for sequences.** The canonical step list for each workflow lives in `workflows/Wn-*.md`. Read it; never hard-code a sequence in this skill.

## When to Use

- You're starting a project and want a guided path from idea to deploy
- You're mid-flight on a workflow and a new chat needs to pick up where you left off
- You've been doing ad-hoc skills and want to step back into a structured sequence
- You want to know "what's next in W4" without reading the workflow doc cold

## When NOT to Use

- One specific task — use the targeted skill directly (`/triage`, `/build-feature`, etc.)
- Don't know what state the project is in — run `/next` first; it answers "where am I?" without committing to a workflow
- Brand new project with no context — run `/start` first, then `/workflow`
- You want to see all skills, not just one workflow — run `/skills`

## The state file: `.claude/workflow-state.md`

This is the pilot's memory. Format follows the repo's `.claude/` convention: **YAML frontmatter for machine-readable state, Markdown body for the human-readable checklist.** The user can edit either side directly; the pilot reconciles on next run.

```markdown
---
workflow_id: W4
workflow_name: Migrate a Prototype to Production
started: 2026-05-13
last_active: 2026-05-13T14:32:00Z
mode: active                 # active | paused | off-script | completed
current_step_index: 3
current_step_slug: /unvibe
next_step_slug: /prd
total_steps: 12
completed_count: 2
---

# Workflow State

## Position
**current:** step 3 of 12 — `/unvibe`
**next:** `/prd` (optional retrofit)

## Sequence

- [x] `/code-map` — completed 2026-05-13
- [x] `/migrate-from-vibe` — completed 2026-05-13
- [>] `/unvibe` — current
- [ ] `/prd` (optional)
- [ ] `/plan`
- [ ] `/setup-database`
- [ ] `/add-auth`
- [ ] `/add-payment` (if needed)
- [ ] `/add-monitoring`
- [ ] `/check-production`
- [ ] `/triage`
- [ ] `/deploy`

## Detours (off-script work)
- 2026-05-13: ran `/triage` for a bug found mid-migration. Returned to sequence.

## Decisions and skips
- 2026-05-13: skipped `/prd` retrofit — user already has a clear product spec.
```

The frontmatter keys are the **source of truth for position** (`current_step_index`, `mode`, `last_active`). The body's checklist is the human-readable view of the same state. When they disagree (e.g., the user manually edited the body), the pilot reconciles: it trusts the body's markers and rewrites the frontmatter to match.

Status markers:
- `[ ]` pending
- `[>]` current — the recommended next step
- `[x]` completed (artifact present or user marked done)
- `[~]` skipped by user choice
- `[!]` blocked / needs attention before proceeding

## Procedure

### Step 1: Determine workflow

Branch on what's present:

**Case A: `workflow-state.md` exists and `mode != completed`.**
Read it. This is a resume. Skip to Step 2.

**Case B: `workflow-state.md` does not exist OR mode is `completed`.**
Ask the user which workflow to start. Show this menu:

> **Which workflow?**
> - **W1** — Prototype a new idea (throwaway clickable mockup)
> - **W2** — Production-grade SaaS (full rigor, end-to-end)
> - **W3** — Add a feature to an existing product
> - **W4** — Migrate a prototype to production (rescue a vibe-coded MVP)
> - **W5** — Refactor & modernize an existing codebase
> - **W6** — Fix a production bug
> - **W7** — Audit & harden for launch
> - **W8** — Build a personal-use tool
>
> Or describe what you're doing and I'll suggest one.
>
> *(See [WORKFLOWS.md](../../WORKFLOWS.md) for the "pick when" cheat sheet.)*

If user describes a goal instead of picking, suggest the best fit with a one-line rationale, then confirm.

**Case C: User passes an explicit argument (e.g. `/workflow W4`).**
Skip the menu; start that workflow.

### Step 2: Build (or resume) the sequence

Read `workflows/Wn-*.md` for the selected workflow. Extract the **"Skill sequence"** section. That section is the source of truth — never hard-code a sequence in this skill.

If a state file already existed (Case A above):

1. Reconcile completed steps with reality. For each `[x]` step that produces a tracked artifact (`.claude/prd.md`, `.claude/plan.md`, `.claude/architecture.md`, `.claude/unvibe-plan.md`, etc.), verify the artifact still exists. If missing, downgrade to `[ ]` and warn the user.
2. Reconcile undetected progress. For each `[ ]` step whose artifact has appeared in `.claude/` (or was logged in `progress.md`) since `last_active`, ask the user: "Looks like `/prd` ran outside the pilot — mark as done?"
3. Update `last_active` timestamp.

If a state file did NOT exist (Cases B / C): write a new `.claude/workflow-state.md` with all steps marked `[ ]` and the first step marked `[>]`.

### Step 3: Show position and recommend next

Output a tight one-screen status:

```
WORKFLOW : W4 — Migrate a Prototype to Production
MODE     : active
PROGRESS : 2 of 12 done

CURRENT  : [>] /unvibe — rehabilitate the leftover mess from /migrate-from-vibe
          produces: .claude/unvibe-plan.md + a series of approved commits

NEXT IN QUEUE
  [ ] /prd (optional retrofit)
  [ ] /plan
  [ ] /setup-database

OPTIONS
  run         — invoke /unvibe now
  skip        — mark skipped, move to next
  done        — mark complete (you ran it outside the pilot)
  pause       — stop here; resume later with /workflow
  back        — step back one
  switch Wn   — switch to a different workflow
  off-script  — go freeform; pilot won't push until you call it back
  exit        — leave the pilot; state file is preserved
```

Wait for the user's response.

### Step 4: Handle the user's response

Translate the user's words to one of the actions. Don't insist on exact syntax — `go`, `yes`, `let's run it`, `do it` all mean **run**. `nope`, `skip this`, `pass` all mean **skip**. Be generous.

| User intent | Action |
|---|---|
| **run** | Tell the user the exact slash command to type (or invoke if they ask you to). Wait for the skill's artifact / completion signal. |
| **skip** | Mark `[~]`, log to "Decisions and skips" with one-line rationale (ask if rationale isn't obvious), advance current to next. |
| **done** | Mark `[x]`, log under "Decisions and skips" if context warrants ("user ran /prd outside pilot"), advance. |
| **pause** | Set `mode: paused`, write a "paused on YYYY-MM-DD at step N" note, append a single progress.md entry, end the turn. |
| **back** | Move `[>]` marker back one position; revert prior step to `[ ]` only if the user explicitly says "redo." Otherwise leave the prior step `[x]` and just move the cursor. |
| **switch Wn** | Archive current state file to `.claude/workflow-state-archived-<workflow>-<date>.md`, append a progress.md entry ("switched W4 → W2"), restart at Step 2 with the new workflow. |
| **off-script** | Set `mode: off-script`. Tell the user: "Go ahead. Run whatever you need. Call `/workflow` again when you want me to pick back up. I'll detect any artifacts you produced and reconcile then." Exit turn. |
| **exit** | Same as pause but more terse. State file preserved. |

After any state-changing action, rewrite `.claude/workflow-state.md`.

### Step 5: Detect completion after a step runs

When the user runs the recommended skill (Step 4 → **run**), wait for the conversation to indicate the skill finished. Detection signals, in priority order:

1. The skill's artifact appears in `.claude/` (e.g., `.claude/prd.md` for `/prd`, `.claude/plan.md` for `/plan`)
2. The user says "done" / "that's done" / "next" or similar
3. A new entry appears in `.claude/progress.md` whose `skill` field matches

When detected:

1. Mark the step `[x]`
2. Move `[>]` to the next pending step (skipping any `[~]` and `[!]` until a `[ ]` is found)
3. Rewrite the state file
4. Resurface Step 3 (show new position)

If multiple steps complete in a single user turn (e.g., the user goes off-script and runs three skills), reconcile in one pass — mark each step `[x]` and log them in "Detours" if they were out of sequence.

## Flexibility — designed for veering off

Workflows are guides, not railroads. The pilot supports four flavors of veering off:

1. **Skipping individual steps.** The user can skip any step with a one-word "skip." The state file records the skip with a timestamp and rationale (asked if not obvious). The pilot does not nag — once skipped, it moves on.

2. **Ad-hoc skills mid-workflow.** Sometimes a bug appears, a dependency breaks, a stakeholder asks for a slide. The user can run any skill at any time. When they return and run `/workflow`, the pilot detects the detour (via `progress.md` entries with no matching sequence step) and logs it under "Detours" — without penalty. The recommended next step doesn't change.

3. **Off-script mode.** The user explicitly says they want to work freeform for a while. The pilot sets `mode: off-script` and does not prompt for the next step. When the user runs `/workflow` again, the pilot picks back up from where the cursor was, reconciling any artifacts that landed in the meantime.

4. **Switching workflows.** The user can switch from W2 to W4 (or any combination) mid-flight. The pilot archives the prior state file (so the user can switch back), starts the new workflow, and logs the switch in `progress.md`.

The user can also edit `.claude/workflow-state.md` directly. The pilot reads the file fresh every run; manual edits are honored.

## New-chat resumption

The pilot must work across context resets. Every invocation:

1. Reads `.claude/workflow-state.md` (the source of truth for position)
2. Reads `.claude/progress.md` (last 10 entries — for detecting detours and out-of-sequence completions)
3. Globs `.claude/` (for artifacts that may indicate completed steps)
4. Reconciles all three into the current view, surfaces any inconsistencies to the user, then resurfaces Step 3

A user opening a fresh chat after two weeks should be able to type `/workflow` and within 30 seconds see exactly where they are, what's next, and what they might have done outside the pilot.

## Rules

- **Never invoke a skill without explicit user confirmation.** Recommend, summarize, and wait.
- **The workflow file is the source of truth.** Read `workflows/Wn-*.md` to get the sequence; do not hard-code it here. If the workflow doc updates, the pilot's recommendations update automatically.
- **The state file is the position of record.** Always rewrite it after a state-changing action. Trust it on resume.
- **Conditional steps stay conditional.** A workflow's `(optional)` or `(if needed)` annotation propagates into the state file's sequence. The pilot still asks for these — but skipping is the default-light option.
- **Don't gatekeep prerequisites.** If a step "should" come before another but the user wants to jump ahead, allow it and note it under "Decisions and skips."
- **Don't auto-resume after pause.** If `mode: paused`, the pilot's first response is "You paused on `<date>` at `<step>`. Want to resume from there, or pick a different starting point?" — never silently continue.
- **Don't overwrite a user's manual edits.** If `.claude/workflow-state.md` was edited by hand (e.g., the user manually marked something `[x]`), respect it. Reconcile, don't override.
- **One screen per status update.** The pilot's surface area is small on purpose. If there's more to say, point at the state file or the workflow doc.

## If something goes wrong

- **State file is corrupted / unreadable** — back it up to `.claude/workflow-state.broken.md`, tell the user, and ask whether to start fresh from the current artifacts or start a new workflow.
- **Workflow doc not found** — fallback: show the user the workflows menu and ask them to pick another. Note in `progress.md` that the requested workflow file was missing.
- **Two workflows seem to be in progress** (state file says W4, but `.claude/` has W2's full artifact set) — surface the discrepancy. Ask the user which is authoritative. Archive the other.
- **A step's artifact is expected but absent after the user said "done"** — politely double-check: "I don't see `.claude/prd.md`. Did `/prd` actually write the artifact, or are we tracking this differently?" Don't auto-mark complete.
- **The user wants to redo a completed step** — confirm, then revert that step to `[ ]`, optionally delete the artifact (with their permission), and move `[>]` back to it.

## Notes for the calling agent

- This skill is designed to be **invoked many times across a project's life**. Every session might start with `/workflow` to re-orient. Keep responses tight.
- The skill never produces a large artifact of its own. The state file is the only output, and it's small by design.
- Pairs naturally with `/next` (workflow-agnostic dashboard) and `/resume` (general-purpose session recovery). Use them when the user hasn't yet picked a workflow; use `/workflow` once they have.
- Compatible with all 8 workflows as defined in `workflows/`. Adding a new workflow only requires creating a new `workflows/W9-*.md` with a "Skill sequence" section — the pilot will pick it up automatically.
