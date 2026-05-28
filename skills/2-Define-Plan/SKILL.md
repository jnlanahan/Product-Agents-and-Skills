---
name: plan
description: MUST BE USED to turn a PRD (or current conversation context) into an executable implementation plan. Breaks work into vertical slices (tracer bullets), assigns a TDD strategy per slice, sequences commits by layer (schema → storage → routes → hooks → components), and writes the plan to `.claude/plan.md` so `/build-feature` can pick it up. Replaces GitHub-Issues-style breakdown. Do NOT use without a PRD or clear requirements — run `/prd` first so the plan has a solid requirements baseline to slice against.
when_to_use: "User says 'create a plan', 'plan this feature', 'how should we implement this', 'break this down', 'make an implementation plan'."
---

# /plan

You turn a PRD (or current conversation context) into an executable plan that `/build-feature` can pick up. The output is `.claude/plan.md` — vertical slices, each demoable end-to-end, each with a TDD strategy.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Before You Start

- A `.claude/prd.md` should exist before planning — if not, run `/prd` first so the plan has a requirements baseline.
- Clarify the desired scope with the user before generating; a plan for a full feature vs. a single route produces very different outputs.

## Inputs

- A `.claude/prd.md` if it exists (preferred starting point)
- Otherwise, the current conversation context
- The codebase (explored on demand to ground decisions)

## Procedure

### Step 1: Detect existing PRD or context

Read `.claude/prd.md` if it exists. If not, synthesize from conversation. If the context is too thin to plan, stop and recommend `/prd` first.

### Step 2: Detect the project

Run in parallel:
- `stack-detector` — framework, db, orm, validation lib, test framework
- `pattern-finder` — "Find a recent feature representative of the project's layering. Trace it through schema → storage → API → client hook → component."

Read `_stack-preferences.md` and `_adaptation-playbook.md`.

### Step 3: Enter plan mode — fast slice sketch

Call `EnterPlanMode` now.

In plan mode, produce a **fast sketch only** — do not write the full plan yet. For each candidate slice, show:

- **Slice title** — short name (e.g., "Tracer bullet — empty list view")
- **One-sentence behavior** — user-facing
- **Layers touched** — schema / storage / route / hook / component (one line)
- **Depends on** — slice number(s) or "none"

Present the sketch as a numbered table. Keep it to one screen. The goal is to get a redirect opportunity before writing 200 lines.

### Step 4: Quiz the user on granularity

While still in plan mode, ask one batch of questions:

- Are these slices the right size, or do any need to be bigger or smaller?
- Are there dependencies I missed (e.g., slice 3 actually needs something from slice 1)?
- What's the priority order if we ship one at a time?

Wait for answers before proceeding.

### Step 5: Elaborate into the full plan

Using the approved slice structure, write the complete plan to `.claude/plan.md` using the WBS template. → See [plan-template.md](references/plan-template.md)

Fill in every section: objectives, success criteria, scope, architecture decisions, and all slice subsections (schema through verification checklist).

### Step 6: Exit plan mode — approval gate

Call `ExitPlanMode`. The user will review the written `.claude/plan.md` and explicitly approve before `/build-feature` picks it up. Do not proceed to post-flight until approved.

### Step 7: Hand off

Tell the user the approved plan is at `.claude/plan.md` and recommend `/build-feature` to start executing the first slice.

## Plan Template

→ See [references/plan-template.md](references/plan-template.md) for the full WBS template Claude writes to `.claude/plan.md`.


## TDD Workflow (apply per slice)

### Philosophy

**Tests verify behavior through public interfaces, not implementation details.** Code can change entirely; tests shouldn't. Good tests survive refactors. Bad tests are coupled to implementation — they break when you rename an internal function.

See `tests.md` for examples and `mocking.md` for mocking guidelines.

### Anti-pattern: horizontal slicing

**DO NOT write all tests first, then all implementation.** This produces:

- Tests for *imagined* behavior, not actual
- Tests that verify shape (signatures, types) instead of user-facing behavior
- Tests insensitive to real changes

**Correct**: vertical slices via tracer bullets. One test → one impl → repeat.

```
WRONG (horizontal):  RED: test1, test2, test3, test4    GREEN: impl1, impl2, impl3, impl4
RIGHT (vertical):    RED→GREEN: test1→impl1   RED→GREEN: test2→impl2   ...
```

### Per-cycle checklist

```
[ ] Test describes behavior, not implementation
[ ] Test uses public interface only
[ ] Test would survive an internal refactor
[ ] Code is minimal for this test
[ ] No speculative features added
```

### Refactor pass

Only **after** all tests in a slice are green. See `refactoring.md` for what to look for: extract duplication, deepen modules (small interface, lots of implementation behind it — see `deep-modules.md`), apply SOLID where natural. Run tests after each refactor step. **Never refactor while RED** — get green first.

## Rules

- **Slices, not layers.** A plan that says "first do all the schema, then all the routes" is a horizontal plan and produces brittle work. Every slice is end-to-end.
- **One test → one impl per cycle.** No batching.
- **Mock only at system boundaries** (DB, third-party APIs, time, randomness). Never internal collaborators. See `mocking.md`.
- **Tests describe behavior**, not implementation. If renaming an internal function would break a test, the test is wrong.
- **Write the plan to `.claude/plan.md`.** No GitHub Issues required. Recommend the user commit it.
- **Hand off to `/build-feature`** at the end. Don't start writing code in this skill.

## If Something Goes Wrong

- **PRD is missing or too vague** — stop and run `/prd` first; a plan built on insufficient requirements produces incorrect slice ordering.
- **Plan produces too many slices** — ask the user to narrow scope; flag which slices are optional vs. required for a first working version.
- **Slice dependencies are circular** — surface the cycle explicitly and ask the user which dependency to break before proceeding.