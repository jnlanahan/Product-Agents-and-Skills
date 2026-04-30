---
name: 0-Setup-Project
description: MUST BE USED when starting a brand-new SaaS project from empty or fresh-scaffold state. Walks the user through third-party account setup, scaffolds the preferred stack in disciplined waves (one commit per wave), generates a CLAUDE.md with a skills index, and seeds git/GitHub conventions. NOT for existing projects — for those, run `/next-steps` first.
---

# /setup-project

You set up a new SaaS project from near-empty state using the user's preferred stack from `_stack-preferences.md`. Discipline matters here: each layer goes in as a separate commit, in the right order, with verification.

## When to Use

- Empty directory or `npm init -y` output
- Fresh `create-next-app` / `create-vite` / `create-t3-app` scaffold with no real features
- User explicitly says "start a new SaaS project"

## When NOT to Use

- Project has source files beyond a scaffold → run `/next-steps` first to assess
- Project has an existing auth/payment/db setup → use the targeted `/add-*` skill instead
- Project came from a vibe-coding tool (Replit/V0/Lovable/Bolt) → run `/migrate-from-vibe` first

## Procedure

### Step 1: Verify greenfield

Run `stack-detector` and `codebase-classifier` in parallel. If classification is **not** `greenfield`:

> Stop. Tell the user: "This project is `<wired|vibe-coded>`. `/setup-project` is greenfield only. Run `/next-steps` to understand what's there, then use targeted `/add-*` skills."

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

Write `CLAUDE.md` at the repo root. Use this exact structure (replace `<...>` placeholders with actual values):

```markdown
# CLAUDE.md — <project name>

This file is read by Claude Code on every conversation. Keep it short.

## Project Overview

<one-paragraph description of what this app does and who it's for>

## Stack

- Framework: <Next.js App Router | React + Vite + Express>
- DB: Neon Postgres + Drizzle ORM
- Auth: Firebase Auth
- Payments: Stripe
- File storage: Firebase Storage
- Monitoring: Sentry (errors) + PostHog (product analytics)
- Tests: <Vitest | Jest>
- Deploy: Railway

## Conventions

- TypeScript strict mode
- Validation at every API boundary using Zod
- Server-side ownership checks on every protected route
- Migrations via `npm run db:generate` then `npm run db:migrate` (never `db:push` against prod)
- One layer per commit when adding features
- Conventional commits: `feat:`, `fix:`, `refactor:`, `chore:`, `docs:`

## What Not to Touch

- `server/migrations/` — never edit applied migrations; create new ones instead
- `firebase-storage.rules` — review carefully, deploy separately
- `.env.example` — keep in sync with actual env vars used; do NOT add real values

## Available Skills

The agent has access to these skills. It will suggest the right one based on what you're doing.

**Setup**
- /next-steps — production-readiness check + what to do next
- /setup-project — scaffold a new SaaS from empty

**Define (turn ideas into specs)**
- /prd — produce a comprehensive PRD from current context
- /plan — turn the PRD into vertical slices with TDD strategy
- /refactor — find or plan a refactor with safe small commits
- /glossary — extract project's domain terms
- /grill-me — stress-test a plan with relentless questions

**Design**
- /prototype — generate 3 clickable HTML design variants

**Build**
- /code-map — explain a code area at a higher level
- /setup-database — wire DB schema + migrations safely
- /migrate-from-vibe — move a Replit/V0/Lovable project to a real stack
- /add-auth, /add-payment, /add-files, /add-monitoring
- /build-feature — implement a feature in TDD layers

**Validate**
- /check-production — full production-readiness audit
- /triage — interactive bug session (root cause + fix plan)

**Deploy**
- /deploy — walk through deployment end-to-end

## Git & GitHub Basics

- Branch naming: `feat/<short-desc>`, `fix/<short-desc>`, `refactor/<short-desc>`
- Commit message format: `<type>: <imperative summary>` (max 72 chars subject line)
- One layer per commit when shipping a feature
- PR title matches the branch's primary commit; description should reference any relevant `.claude/plan.md` or `.claude/refactor-plan.md`
- Never `git push --force` to `main`; force-push only your own feature branches if needed

## Tests

- `npm test` — run the suite
- Tests verify behavior through public interfaces (API responses, UI state) — not internals
- Mock at system boundaries (DB, third-party APIs, time, randomness) — never internal collaborators
```

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
> - Run `/next-steps` anytime to see project state
> - When ready to ship: `/check-production` then `/deploy`

## Rules

- **External accounts before code.** Don't install anything until the user has the credentials in hand.
- **One wave per commit.** Easier review, easier rollback.
- **Verify after each wave.** Don't proceed if `npm run build` is broken.
- **Don't write app features.** This skill wires infrastructure. The first user-facing feature is `/build-feature`.
- **Don't deploy.** That's `/deploy`.
- **Don't run `db:push` against a production database.** Local dev only. Production uses `db:migrate`.
- **Always generate `CLAUDE.md` with the skills index.** Without it, the agent can't suggest commands the user has forgotten.
