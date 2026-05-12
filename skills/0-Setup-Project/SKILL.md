---
name: setup-project
description: MUST BE USED when starting a brand-new SaaS project from empty or fresh-scaffold state. Walks the user through third-party account setup, scaffolds the preferred stack in disciplined waves (one commit per wave), generates a CLAUDE.md with a skills index, and seeds git/GitHub conventions. Use `--personal` flag for lighter personal tools (no Stripe, optional auth, SQLite). NOT for existing projects — for those, run `/next` first.
---

# /setup-project

You set up a new project from near-empty state using the user's preferred stack from `_stack-preferences.md`. Discipline matters here: each layer goes in as a separate commit, in the right order, with verification.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Only run on an empty directory or fresh scaffold — this skill scaffolds opinionated structure and will overwrite conflicting files.
- For existing projects, run `/next` first; do NOT run `/setup-project` on a project that already has real code.
- Verify the target directory before starting (`ls`) and confirm it is empty or contains only a README or basic scaffold.

## When to Use

- Empty directory or `npm init -y` output
- Fresh `create-next-app` / `create-vite` / `create-t3-app` scaffold with no real features
- User explicitly says "start a new SaaS project" or "start a personal tool"

## When NOT to Use

- Project has source files beyond a scaffold → run `/next` first to assess
- Project has an existing auth/payment/db setup → use the targeted `/add-*` skill instead
- Project came from a vibe-coding tool (Replit/V0/Lovable/Bolt) → run `/migrate-from-vibe` first

## `--personal` Mode (lighter stack for personal tools)

When invoked as `/setup-project --personal`, use a reduced scope:

| Layer | SaaS default | Personal (`--personal`) |
|---|---|---|
| DB | Neon Postgres + Drizzle | SQLite + Drizzle (local-first, no cloud account needed) |
| Auth | Firebase Auth (required) | Optional — ask if needed |
| Payments | Stripe | Skip |
| File storage | Firebase Storage | Skip |
| Monitoring | PostHog + Sentry | Sentry only (optional) |
| Deploy | Render / Railway | Fly.io or Render (single container) |

Everything else (test infra, CI, CLAUDE.md, security middleware) still applies. Use `--personal` for: internal dashboards, scripts with a UI, personal productivity tools, side projects without a business model.

Skip the "External account setup" step for omitted services. Ask only about the accounts that are actually needed.

## Procedure

### Step 1: Verify greenfield

Run `stack-detector` and `codebase-classifier` in parallel. If classification is **not** `greenfield`:

> Stop. Tell the user: "This project is `<wired|vibe-coded>`. `/setup-project` is greenfield only. Run `/next` to understand what's there, then use targeted `/add-*` skills."

### Step 2: Confirm framework

Read `_stack-preferences.md`. Ask:

> Two architecture options:
> 1. **Next.js App Router** (single codebase; recommended for new SaaS)
> 2. **React + Vite + Express** (split client/server; matches the reference template)
>
> Which?

### Step 3: Confirm scope

Ask which integrations to wire now (defaults all on):

- [x] Drizzle + Neon (almost always needed)
- [x] Firebase Auth (almost always needed)
- [x] Stripe (defer if pre-revenue)
- [x] Firebase Storage (defer if no file uploads)
- [x] PostHog + Sentry (wire from day one)
- [ ] AI SDK (only if AI features planned)

User can opt out of any.

### Step 4: External account setup (USER does these in browser FIRST)

Before installing any code, the user needs accounts and credentials. Tell them, verbatim:

> **You need these accounts before I install anything. Open each, sign up, and grab the values listed:**
>
> 1. **Neon** (https://neon.tech) — sign up free → create project → copy `DATABASE_URL` from Connection Details
> 2. **Firebase** (https://console.firebase.google.com) — create project → enable Authentication (Email + Google providers) → Project Settings → General → SDK config (copy 6 `NEXT_PUBLIC_FIREBASE_*` values) → Service Accounts tab → "Generate new private key" (saves a JSON file — you'll paste the FULL content)
> 3. **Stripe** (https://dashboard.stripe.com/register) — sign up → leave in TEST mode for now → Developers → API keys → copy `pk_test_*` and `sk_test_*`
> 4. **PostHog** (https://posthog.com) — sign up → copy Project API Key (`phc_*`) and Host URL
> 5. **Sentry** (https://sentry.io) — sign up → create project (pick your framework) → copy DSN → Settings → Auth Tokens → create with `project:releases` scope
> 6. **GitHub** (https://github.com) — sign up if needed; you'll create a repo for this project at the end of setup
>
> Reply with all the values OR "I have them in a password manager — let's go."

Wait for the user.

### Step 5: Show the install plan

Show the full list of dependencies, files to create, files to modify, env vars needed. Get explicit approval before any `npm install`.

### Step 6: Execute in waves (one commit per wave)

Each wave is a single commit — easier to roll back, easier to debug.

**Wave 1: Framework + TypeScript**
- `create-next-app` if not done; ensure tsconfig strict
- Commit: "scaffold framework"

**Wave 2: DB + ORM**
- Drizzle config, schema with `users` table, first migration
- Commit: "wire drizzle + neon"

**Wave 3: Auth**
- Firebase client + admin SDK, route protection middleware
- Commit: "wire firebase auth"

**Wave 4: Payments**
- Stripe client, Checkout endpoint, webhook handler with signature verification, Customer Portal endpoint
- Commit: "wire stripe checkout + webhook"

**Wave 5: Storage**
- Firebase Storage client, upload route, security rules
- Commit: "wire firebase storage"

**Wave 6: Monitoring** (delegate to `/add-monitoring` if you want)
- PostHog (client + server), Sentry (client + server), wired into layout + error boundaries
- Commit: "wire posthog + sentry"

**Wave 7: AI** (only if requested) — delegate to project owner; skip auto-scaffolding
- Commit: "wire ai sdk"

**Wave 8: Security middleware**
- Rate limiting, CORS, headers
- Commit: "add security middleware"

**Wave 9: Test infrastructure**
- Install test framework (Vitest for Next.js, Jest for Vite/Express)
- Add `npm test`, `npm run test:watch` scripts
- Add a single passing smoke test so the suite isn't empty
- Commit: "add test infrastructure"

**Wave 10: CI**
- `.github/workflows/test.yml` with typecheck + tests on PR
- Commit: "add CI workflow"

**Wave 11: Health check + docs**
- `/api/health` endpoint, complete `.env.example`
- README with quick-start
- **Generate `CLAUDE.md` with the skills index** (see Step 7 below)
- Commit: "add health check, env example, and CLAUDE.md skills index"

**Wave 12: Initialize git + GitHub repo**
- Confirm `.gitignore` covers `.env`, `node_modules`, build outputs, `.claude/` (except committed planning docs)
- `git init` if not already done
- Make the initial commit if waves 1-11 weren't already committed
- Ask user if they want a GitHub repo created now; if yes, walk through `gh repo create` (or browser flow)
- Commit: (no new code; just push to remote if applicable)

After each wave: `npm run build && npm run check`. If broken, fix before next wave.

### Step 7: Generate CLAUDE.md with skills index

→ See [claude-md-template.md](references/claude-md-template.md) for the exact `CLAUDE.md` structure to write at the repo root. Fill in the `<...>` placeholders with actual project values.

### Step 8: End-to-end verification

- [ ] `npm run dev` boots without errors
- [ ] `/api/health` returns 200
- [ ] Sign up + sign in flow works (visit a protected page)
- [ ] Stripe Checkout test card succeeds (4242 4242 4242 4242)
- [ ] `stripe listen --forward-to localhost:3000/api/webhooks/stripe` shows webhook events delivered
- [ ] PostHog event appears in dashboard
- [ ] Sentry test exception appears in dashboard
- [ ] CI passes on first PR
- [ ] `CLAUDE.md` exists at repo root and includes the Available Skills section

### Step 9: Hand off

> Project is ready for development. Next:
> - Run `/prd` to produce a PRD for your first feature
> - Then `/plan` to turn it into vertical slices
> - Then `/build-feature` to implement
> - Run `/next` anytime to see project state
> - When ready to ship: `/check-production` then `/deploy`

## Rules

- **External accounts before code.** Don't install anything until the user has the credentials in hand.
- **One wave per commit.** Easier review, easier rollback.
- **Verify after each wave.** Don't proceed if `npm run build` is broken.
- **Don't write app features.** This skill wires infrastructure. The first user-facing feature is `/build-feature`.
- **Don't deploy.** That's `/deploy`.
- **Don't run `db:push` against a production database.** Local dev only. Production uses `db:migrate`.
- **Always generate `CLAUDE.md` with the skills index.** Without it, the agent can't suggest commands the user has forgotten.

## If Something Goes Wrong

- **Package install fails** — check Node/pnpm version matches the scaffold requirements; delete `node_modules` and lockfile, then retry.
- **First migration fails** — confirm the database connection string is correct and the database server is running before re-running the migration.
- **Account setup step blocked** (Vercel, Supabase, etc.) — skip the step, continue with local setup, and revisit the account setup separately; do not abort the whole scaffold.
- **Git push rejected** — confirm the remote is set correctly (`git remote -v`) and you have push permission to the repo.