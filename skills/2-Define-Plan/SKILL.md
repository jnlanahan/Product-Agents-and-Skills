---
name: 2-Define-Plan
description: MUST BE USED to turn a PRD (or current conversation context) into an executable implementation plan. Breaks work into vertical slices (tracer bullets), assigns a TDD strategy per slice, sequences commits by layer (schema → storage → routes → hooks → components), and writes the plan to `.claude/plan.md` so `/build-feature` can pick it up. Replaces GitHub-Issues-style breakdown.
---

# /plan

You turn a PRD (or current conversation context) into an executable plan that `/build-feature` can pick up. The output is `.claude/plan.md` — vertical slices, each demoable end-to-end, each with a TDD strategy.

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

### Step 3: Sketch vertical slices

A **vertical slice** is a tracer-bullet path through every layer for one piece of behavior. It's demoable end-to-end. It is NOT "schema first, then API, then UI" — that's a horizontal slice and produces tests/code that don't match observed behavior.

For each candidate slice, write:

- **Behavior**: a one-sentence description in user-facing language
- **Why this is a slice (not bigger or smaller)**: what makes it a coherent unit
- **Layers touched**: schema, storage, route/action, hook, component
- **Migration**: needed yes/no, sketch the shape
- **Side effects**: emails, webhooks, third-party calls, AI calls
- **TDD strategy**: which behaviors get a test, what the test exercises, what gets mocked (only at system boundaries — see `mocking.md`)

### Step 4: Quiz the user on granularity

Ask the user one batch of questions:

- Are these slices the right size, or do any need to be bigger or smaller?
- Are there dependencies I missed (e.g., slice 3 actually needs something from slice 1)?
- What's the priority order if we ship one at a time?

Wait for answers before writing the plan.

### Step 5: Write the plan to `.claude/plan.md`

Use the template below. Each slice section is independently executable.

### Step 6: Hand off

Tell the user the plan is at `.claude/plan.md` and recommend `/build-feature` to start executing the first slice.

## Plan Template

```markdown
# Implementation Plan: <Feature or Product Name>

*Status: Ready for build · Source: `.claude/prd.md` (or "current conversation") · Last updated: <YYYY-MM-DD>*

## Strategy

- **Approach**: vertical slices, one slice per PR (or per coherent commit chain)
- **TDD**: red → green → refactor, per layer; tests verify behavior through public interfaces, not internals (see TDD Workflow below)
- **Layering** (as detected in this project): <schema → storage → routes → hooks → components | server actions → server components → client components | other>
- **Test mocking boundary**: <DB, third-party APIs, time, randomness> — see `mocking.md`

## Slice Order

| # | Slice | Depends on | Demoable outcome |
|---|---|---|---|
| 1 | <Tracer bullet — minimal happy path> | None | <e.g., user can see an empty list> |
| 2 | <Add primary write path> | 1 | <e.g., user can create one item> |
| 3 | <Add update / delete> | 2 | <e.g., user can update or remove items> |
| 4 | <Edge cases / permissions> | 3 | <e.g., other users can't see your items> |
| 5 | <Polish / non-happy path> | 4 | <e.g., empty states, errors, validation> |

## Slice 1: <Slice Name>

**User-facing behavior**: <one sentence>

**Layers**:

- Schema: <new table/columns or "no change">
- Storage: <new functions or "no change">
- Routes/Actions: <new endpoints or "no change">
- Client: <new hook + component or "no change">

**TDD cycles** (one test → one impl, NOT all tests first):

1. **RED**: test that <observable behavior 1> — should fail because <reason>
   **GREEN**: minimal impl — <one-line description>
2. **RED**: test that <observable behavior 2> — should fail
   **GREEN**: minimal impl — <one-line description>
3. **RED**: test that <ownership/permission boundary> — should fail
   **GREEN**: minimal impl — <one-line description>

**Refactor pass after green**: <e.g., "extract helper if duplication appears"; or "none expected">

**Migration**: <yes/no>; if yes — <sketch>

**Commits**: one per layer (schema, storage, routes, client) — <count> total

**Verification**:

- [ ] All tests green
- [ ] `npm run build` clean
- [ ] Manual: <observable demo step>

**Out of scope for this slice**: <things explicitly deferred to later slices>

## Slice 2: ...

(repeat structure)

## Cross-Cutting Decisions

- **Validation**: <e.g., Zod at every API boundary, `drizzle-zod` for DB shape>
- **Auth/ownership**: <e.g., every protected route checks `userId === resource.userId` server-side>
- **Error handling**: <match existing pattern from `pattern-finder` — e.g., `throw new AppError(...)`>
- **Naming conventions**: <follow existing — e.g., `useNotes`, `noteRoutes.ts`>

## Open Questions

Anything that needs an answer before the relevant slice is buildable:

- <question>

## Out of Scope

Explicit list of things NOT in this plan (mirrored from PRD, plus anything pruned during planning):

- <item>
```

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
