---
title: "CLAUDE.md Template"
skill: "0-Setup-Project"
---

# CLAUDE.md Template

Write this file at the repo root, replacing `<...>` placeholders with actual project values.

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
