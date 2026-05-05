---
title: "Layered Plan Examples"
skill: "4-Build-Feature"
---

# Layered Plan Examples

These examples show the concrete layered plan format for both React+Express and Next.js App Router architectures. Use `pattern-finder` output to determine which pattern matches the current project.

## React + Express (SaaS template)

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

## Next.js App Router

```
LAYERED PLAN: Add "Notes" feature (Next.js App Router)
======================================================

Layer 1: SCHEMA (db/schema.ts + drizzle migration)
  - notes table: id (serial pk), userId (fk users.id), title (text), body (text), createdAt, updatedAt
  - Index on userId for list queries
  - drizzle-zod insertNoteSchema, updateNoteSchema

Layer 2: SERVER (db queries + Server Actions)
  - app/(authed)/notes/_actions.ts:
    createNote(formData), updateNote(id, formData), deleteNote(id)
  - These use 'use server' directive
  - Auth via auth() helper at top of each action

Layer 3: PAGES + COMPONENTS
  - app/(authed)/notes/page.tsx — server component, fetches notes via direct db call
  - app/(authed)/notes/[id]/page.tsx — single note view
  - app/(authed)/notes/new/page.tsx — create form
  - components/notes/NoteEditor.tsx — client component, uses Server Action
```
