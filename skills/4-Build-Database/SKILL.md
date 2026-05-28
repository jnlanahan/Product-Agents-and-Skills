---
name: setup-database
description: MUST BE USED when the user wants to set up a database, add a table, add a column, add an index, or run a migration. Detects ORM (Drizzle / Prisma / Kysely / raw SQL) and adapts. Walks through migration generation → review → apply → verification, with explicit warnings around destructive migrations. Aimed at amateur users — never silently runs destructive operations.
when_to_use: "User says 'add a table', 'add a column', 'run a migration', 'set up the database', 'add an index', 'create a schema'."
---

# /setup-database

Database setup and migration helper. Two main jobs:

1. **First-time setup** — wire a fresh DB connection, ORM, and migration system
2. **Add or change schema** — generate a migration, let the user review it, apply it safely

Aimed at users who haven't run database migrations many times before. Errs heavily on the side of "stop and ask" before anything destructive.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Never run a destructive migration (DROP TABLE, DROP COLUMN, TRUNCATE) without explicit user confirmation and a clear backup strategy.
- Always show the user the generated SQL before applying it — stop and ask if anything looks unexpected.
- On production databases, require the user to confirm the target environment before any `migrate` command runs.

## When to Use

- Brand new project with no DB → wire connection + ORM + first migration
- Adding a new table
- Adding a column to an existing table
- Adding an index, foreign key, or constraint
- Adding seed data
- Investigating a broken migration

## When NOT to Use

- Production deploy (use `/deploy`, which handles `db:migrate` against prod)
- Schema design from scratch — use `/plan` to think through the schema, then come here to apply it

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — what's the framework, what ORM (if any) is in `package.json`
- `pattern-finder` — "Find existing schema files, migration files, and DB connection setup"

Read the project's `CLAUDE.md` for any database-related conventions.

### Step 2: Determine mode

Three cases:

**Case A: No DB yet**
- Confirm DB is Neon Postgres (this is the required default — no alternatives for new projects)
- Confirm ORM is Drizzle (required default)
- Run first-time setup (Step 3)

**Case B: DB and ORM present, user wants to change schema**
- Identify what they want to add (table, column, index, etc.)
- Generate migration (Step 4)

**Case C: User reports a broken migration**
- Read the migration files in order
- Look at `__drizzle_migrations` (or equivalent) table to see which ran
- Diagnose; do NOT auto-fix without showing the user

### Step 3: First-time setup

Walk through:

1. **Get the connection string** from the user (Neon dashboard → Connection Details → copy `DATABASE_URL`).
2. **Add `DATABASE_URL` to `.env.local` and `.env.example`** (the example without the actual value).
3. **Wire t3-env** to validate `DATABASE_URL` at startup — the app should crash loudly at boot if the DB string is missing, not silently at query time.
4. **Install Drizzle**: `npm install drizzle-orm @neondatabase/serverless` and `npm install -D drizzle-kit`.
4. **Create `drizzle.config.ts`** at the repo root.
5. **Create `shared/schema.ts`** (or `db/schema.ts` for Next.js projects) with one starter table — `users` typically.
6. **Add npm scripts**: `db:generate`, `db:migrate`, `db:push`, `db:studio`, `db:check`.
7. **Generate the first migration**: `npm run db:generate`. Confirm a `.sql` file appears under `server/migrations/` (or `db/migrations/` or `drizzle/`).
8. **Review the SQL** with the user — read it together, explain what each statement does.
9. **Apply it**: `npm run db:migrate`. Confirm success (no errors, table exists).
10. **Verify**: open Drizzle Studio (`npm run db:studio`), confirm the table is there.
11. **Commit**: `feat: wire drizzle + neon`.

### Step 4: Generate a migration for a change

For any schema change:

1. **Edit the schema file** (`shared/schema.ts` or `db/schema.ts`) — add the column / table / index / constraint.
2. **Run `npm run db:generate`** — Drizzle Kit produces a new SQL file under `server/migrations/`.
3. **Show the user the generated SQL.** Read it line by line. Highlight anything destructive.
4. **Stop and ask before applying** if the migration is destructive (see Destructive Operations below).
5. **Apply with `npm run db:migrate`** — verify no errors.
6. **Verify the change in Drizzle Studio** — confirm the new column / table / etc. is there.
7. **Commit the schema change AND the migration file together** — one commit. Never commit a schema change without its migration file.

### Step 5: Verify

After any migration:

- [ ] `npm run db:generate` produces no new files (you generated everything you intended)
- [ ] `npm run db:migrate` exits successfully
- [ ] `npm run db:check` reports no inconsistencies
- [ ] Drizzle Studio shows the expected schema
- [ ] Tests still pass (`npm test`)
- [ ] Schema file and migration file are in the same commit

## Destructive Operations — Always Stop and Ask

**Stop and ask the user before running any of these.** Show the SQL, explain what it does, and only proceed on explicit "yes":

- `DROP TABLE`
- `DROP COLUMN`
- `ALTER TABLE ... ALTER COLUMN ... TYPE` (column type change can lose data)
- `RENAME COLUMN` (breaks code that still references the old name)
- Removing a NOT NULL constraint (probably fine; double-check anyway)
- Removing a unique constraint or primary key
- Anything against a column that already has data

If the user is doing this in production via `/deploy`, the deploy skill takes over — never use `db:push` against production. Use `db:migrate` only.

## ORM-Specific Notes

### Drizzle (preferred)

- Schema lives in `shared/schema.ts` or `db/schema.ts`
- `db:generate` creates SQL migrations from schema diff
- `db:migrate` applies pending migrations (uses `__drizzle_migrations` table to track)
- `db:push` skips migrations and pushes schema directly — **dev only, never prod**
- `db:studio` is a UI for browsing data
- `drizzle-zod` produces Zod validators from your schema — use them at API boundaries

### Prisma (if detected)

- Schema lives in `prisma/schema.prisma`
- `npx prisma migrate dev` generates and applies in dev
- `npx prisma migrate deploy` applies in production (no generation)
- `npx prisma studio` is the UI
- Prisma Client is regenerated on every `migrate` — make sure CI runs `prisma generate` before build

### Raw SQL (if detected)

- Migrations are likely hand-written SQL files in a `migrations/` folder
- Look for an existing runner (`node-pg-migrate`, custom script, etc.) — mirror its conventions
- If there's no runner, ask the user before introducing one

## Common Mistakes to Avoid

- **Editing an applied migration file.** Once a migration has run, it's frozen. Add a new migration to fix mistakes.
- **Committing `db:push` changes to production.** It bypasses the migrations system; production state diverges from migration history. Always `db:generate` + `db:migrate`.
- **Dropping a column without first removing the code that references it.** Stop, deprecate the code, ship that, then drop the column.
- **Adding a NOT NULL column without a default to an existing table with rows.** The migration will fail. Either add with a default, or do a 3-step migration: add nullable → backfill → set NOT NULL.
- **Deleting a foreign key that other tables still reference.** Migration fails. Drop dependent rows or constraints first.
- **Mixing schema changes and seed data in the same migration.** Schema migrations and data migrations are different concerns; separate them.

## Rules

- **Always show the generated SQL before applying.** No silent migrations.
- **Always commit schema and migration together.** Never separately.
- **Never `db:push` against production.** Local dev only.
- **Always stop and ask before destructive operations.** Drops, type changes, renames, NOT NULL on existing columns.
- **Always run `npm run db:check` after a migration** — it catches ORM/DB drift early.
- **For first-time setup, use Neon + Drizzle — always.** No alternatives for new projects.

## If Something Goes Wrong

- **Migration fails to apply** — read the full error message; common causes are a constraint violation, a column that already exists, or a connection refused. Fix the migration file and re-run; never skip or force.
- **ORM not detected** — check `package.json` for `drizzle-orm`, `@prisma/client`, or `kysely`; if absent, ask the user which ORM to install before proceeding.
- **Connection string invalid** — test the connection with `npx prisma db pull` (Prisma) or `drizzle-kit introspect` (Drizzle) to get a clear error before proceeding.
- **Destructive migration rolled back unexpectedly** — do not retry without reading the database logs; partial migration rollback can leave the schema in an inconsistent state.