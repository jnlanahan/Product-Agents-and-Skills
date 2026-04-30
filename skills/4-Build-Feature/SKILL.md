---
name: 4-Build-Feature
description: MUST BE USED to implement a new feature in coherent TDD layers (schema → storage → routes → hooks → components). Reads `.claude/plan.md` if it exists; otherwise interviews the user. Adapts to the project's actual layering, mirrors existing patterns, and ships one commit per layer with tests at each layer. Trigger on `/build-feature`, "build this feature", "implement this", "let's start coding".
---

# /build-feature

You plan and implement a feature in coherent layers, with tests at each layer. Adapts to whatever architecture the project has. If `.claude/plan.md` exists, execute against it. Otherwise interview the user briefly, then proceed.

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

Use the project's actual layering (from `pattern-finder`), not a generic one. Example for the SaaS template:

```
LAYERED PLAN: Add "Notes" feature
==================================

Layer 1: SCHEMA (shared/schema.ts + migration)
  - notes table: id (serial pk), userId (fk users.id), title (text), body (text), createdAt, updatedAt
  - Index on userId for list queries
  - drizzle-zod insertNoteSchema, updateNoteSchema

Layer 2: STORAGE (server/storage/NoteStorage.ts)
  - createNote(userId, input): returns Note
  - listNotesByUser(userId): returns Note[]
  - getNoteByIdForUser(noteId, userId): returns Note | null  // ownership baked in
  - updateNote(noteId, userId, input): returns Note | null
  - deleteNote(noteId, userId): returns boolean
  - Tests: unit tests with mocked db (existing pattern in __tests__)

Layer 3: ROUTES (server/routes/noteRoutes.ts)
  - GET /api/notes — list current user's notes
  - POST /api/notes — create
  - GET /api/notes/:id — get one (storage call enforces ownership)
  - PATCH /api/notes/:id — update
  - DELETE /api/notes/:id — delete
  - All routes use existing requireAuth + Zod validation patterns
  - Tests: supertest tests with mocked storage (existing pattern)

Layer 4: HOOKS (client/src/hooks/useNotes.ts)
  - useNotes() — list query
  - useNote(id) — single query
  - useCreateNote() — mutation
  - useUpdateNote() — mutation
  - useDeleteNote() — mutation
  - Use existing React Query patterns from useFiles.ts

Layer 5: COMPONENTS (client/src/components/notes/)
  - NoteList.tsx — list view
  - NoteCard.tsx — single item display
  - NoteEditor.tsx — create/edit form (use existing react-hook-form + Zod pattern)
  - Mount under client/src/pages/notes.tsx

Migration needed: Yes — npm run db:generate then db:migrate
External deps to add: None
Risk areas: None — pure CRUD
Estimated commits: 5 (one per layer)
```

For Next.js project, layering looks different:

```
LAYERED PLAN: Add "Notes" feature (Next.js App Router)
======================================================

Layer 1: SCHEMA (db/schema.ts + drizzle migration)
  (same as above)

Layer 2: SERVER (db queries + Server Actions)
  - app/(authed)/notes/_actions.ts:
    createNote(formData), updateNote(id, formData), deleteNote(id)
  - These use `'use server'` directive
  - Auth via auth() helper at top of each action

Layer 3: PAGES + COMPONENTS
  - app/(authed)/notes/page.tsx — server component, fetches notes via direct db call
  - app/(authed)/notes/[id]/page.tsx — single note view
  - app/(authed)/notes/new/page.tsx — create form
  - components/notes/NoteEditor.tsx — client component, uses Server Action
```

### Step 4: Get approval

Show the plan. Wait for "yes" or modifications.

### Step 5: Execute layer by layer

One commit per layer:

1. Schema commit — schema change, migration file, generated types
2. Storage/server commit — data layer + tests
3. Routes/actions commit — API surface + tests
4. Hooks commit (if needed) — data fetching + mutations
5. Components commit — UI

After each commit: `npm run check && npm test` (or whatever the project uses).

### Step 6: Verify end-to-end

Manually exercise the full feature:
- Create a record
- List it
- Update it
- Delete it
- Try to access another user's record → should 403/404
- Test edge case (empty list, very long title, special characters)

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
