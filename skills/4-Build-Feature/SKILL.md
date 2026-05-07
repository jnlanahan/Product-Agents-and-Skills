---
name: build-feature
description: MUST BE USED to implement a new feature in coherent TDD layers (schema → storage → routes → hooks → components). Reads `.claude/plan.md` if it exists; otherwise interviews the user. Adapts to the project's actual layering, mirrors existing patterns, and ships one commit per layer with tests at each layer. Trigger on `/build-feature`, "build this feature", "implement this", "let's start coding".
---

# /build-feature

You plan and implement a feature in coherent layers, with tests at each layer. Adapts to whatever architecture the project has. If `.claude/plan.md` exists, execute against it. Otherwise interview the user briefly, then proceed.

## Important

- Run `pattern-finder` before writing any new file — match the project's naming, import, and error-handling conventions exactly.
- If `.claude/plan.md` exists, execute against it; do not re-plan unless the user explicitly asks for a scope change.
- Ship one commit per layer; do not bundle schema + routes + components into a single commit.

## When to Use

- Any feature that touches more than one of: DB, API, client UI
- Examples: "add a notes feature", "add team invites", "add a usage dashboard"

## When NOT to Use

- One-line bug fixes → just fix them
- Pure UI tweaks (color, copy, layout) → just edit
- Refactors that don't change behavior → use `/refactor`
- Bugs requiring investigation → use `/triage`

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — framework, db, orm, validation lib, test framework
- `codebase-classifier` — wired vs vibe-coded affects how strictly you mirror patterns
- `pattern-finder` — "Find a recent feature that's representative of this project's layering. Trace it through schema → storage/db → API route → client hook → component."

Also: read `.claude/plan.md` if it exists. If yes, the slices and TDD strategy are pre-decided — skip Step 2 and execute slice-by-slice.

### Step 2: Understand the feature (only if no plan exists)

If no `.claude/plan.md`, ask the user enough to plan. Don't guess. Required:

- **What**: one-sentence description
- **Who**: which users see it (all, paid, admin, owner-only)
- **Where**: where in the UI does it appear
- **Data**: what's stored (table sketch)
- **Permissions**: who can read, who can write
- **Side effects**: any emails, webhooks, third-party calls, AI calls

If the feature is non-trivial, recommend the user run `/prd` then `/plan` first, and offer to do that instead.

### Step 3: Write the layered plan

Use the project's actual layering (from `pattern-finder`), not a generic one.

→ See [layered-plan-examples.md](references/layered-plan-examples.md) for a complete React+Express example and a Next.js App Router example, showing the expected plan format including layers, file paths, migration notes, and estimated commits.

### Step 4: Get approval

Show the plan. Wait for "yes" or modifications.

### Step 5: Execute layer by layer

One commit per layer:

1. Schema commit — schema change, migration file, generated types
2. Storage/server commit — data layer + tests
3. Routes/actions commit — API surface + tests
4. Hooks commit (if needed) — data fetching + mutations
5. Components commit — UI

After each commit: `npm run check && npm test` (or whatever the project uses). Then emit a handoff block before moving to the next layer:

```
LAYER HANDOFF
=============
layer_completed : <schema | storage | routes | hooks | components>
files_changed   : <list of created/modified files>
tests_passing   : <yes | no>
build_clean     : <yes | no>
issues_found    : <any unexpected discoveries, or "none">
next_layer      : <what comes next and its key files>
```

Do not proceed to the next layer if `tests_passing: no` or `build_clean: no`.

### Step 6: Verify end-to-end

Manually exercise the full feature:
- Create a record
- List it
- Update it
- Delete it
- Try to access another user's record → should 403/404
- Test edge case (empty list, very long title, special characters)

### Step 7: Independent verification

After all slices are done and end-to-end verification passes:

1. Ask the user to run `/check-production --lite` against the branch. This invokes `prod-readiness-auditor` (a separate agent with no memory of the build context) to flag anything the implementation missed.
2. If a `.claude/plan.md` Validation Contract exists, walk through each contract assertion row-by-row with the user — do not read the implementation to justify failures; just check if each observable outcome is true.
3. Only declare the feature complete when the contract passes and `--lite` shows no Critical or High findings related to the new feature.

This step exists because the implementing agent has success bias — a fresh agent or the user running the validation contract catches errors the implementer explains away.

## TDD Approach (vertical slices, one test → one impl)

For each slice, write the failing test BEFORE the implementation. The cycle:

1. Write the test — what's the expected behavior?
2. Run it — confirm it fails for the right reason
3. Write minimal implementation to pass
4. Run again — confirm it passes
5. Move to the next behavior in this slice
6. After all behaviors in the slice are green, refactor if needed
7. Run all tests after each refactor step
8. Move to next slice

**Anti-pattern**: writing all tests first, then all implementation. That's a horizontal slice and produces tests that verify imagined behavior. Vertical slices (one test → one impl) produce tests sensitive to real changes.

**Mock at system boundaries only** — DB, third-party APIs, time, randomness. Never mock internal collaborators (the storage helper your route calls, etc.). See `../2-Define-Plan/mocking.md` and `../2-Define-Plan/tests.md` for examples.

This is genuinely faster than "code first, test later" once you're used to it. It also catches API design mistakes when they're cheap to fix.

## Adapting to Vibe-Coded Apps

For vibe-coded projects, `pattern-finder` may report inconsistencies. Strategy:

- **If the project has *no* tests**: don't bolt tests onto your new feature only — that creates an island of TDD in a sea of untested code, which is annoying. Instead: surface the gap to the user ("this codebase has no test infrastructure; want me to set up Vitest first?") and let them decide.
- **If the project has *some* tests**: write tests for your new feature; mirror the existing test style.
- **If the project has *inconsistent* layering**: pick the most-recent feature and mirror it. Note the inconsistency.

## Common Pitfalls

- **Adding a feature that breaks an existing one** — run the existing test suite after each commit
- **Wide migrations** — if your migration touches columns other features use, do a separate migration first
- **Skipping the auth layer** — every new route needs auth + ownership checks
- **Inventing a new component library** — use whatever's already in `components/ui/`
- **Inline DB queries in components** in non-Next.js apps — go through the storage layer
- **`any` type creep** — if you add `: any`, you've lost the plot; use Zod-derived types

## Rules

- **Mirror the project, not the SaaS template.** The template's pattern is a reference, not a mandate.
- **One layer per commit.** Easier to review, easier to revert.
- **Tests at each layer.** Even one test per layer is better than no tests.
- **Ownership enforced server-side.** Don't trust the client's claim about who they are.
- **Migration files committed.** Never run `db:push` against production; always `db:migrate`.

## If Something Goes Wrong

- **pattern-finder finds no matching pattern** — the feature type may be novel for this codebase; ask the user to point to the closest existing file to use as a reference.
- **Tests fail after implementation** — run tests in isolation before running the full suite; a failing setup file or shared mock state causes false failures across tests.
- **Build errors after adding new layer** — confirm all imports are correct and types match; do not proceed to the next layer until the current layer compiles cleanly.
- **`.claude/plan.md` scope is incorrect** — stop and run `/plan` again with the corrected scope rather than adapting mid-implementation; diverging from the plan compounds errors.