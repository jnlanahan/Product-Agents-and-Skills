---
name: _adaptation-playbook
description: Reference data — rules every skill follows when deciding whether to install fresh, extend existing, or stay out of the way. Critical for handling vibe-coded apps without breaking them. Not invocable directly.
---

# Adaptation Playbook

Every skill operates on one of three classes of project. The `codebase-classifier` agent decides which:

1. **Greenfield** — empty repo or near-empty (only `package.json`, `README`, `.gitignore`)
2. **Wired** — has clear, consistent patterns
3. **Vibe-coded** — partially built, inconsistent patterns, missing pieces, possibly insecure

## The Five Rules (apply in order)

### 1. Detect before you write

Every skill **must** delegate to `stack-detector` (and usually `pattern-finder`) before making any change. No skill assumes Express, Next.js, Drizzle, Firebase, etc.

### 2. Existing patterns win over preferences

If the project uses Prisma and `_stack-preferences.md` says Drizzle: **use Prisma**. If the project uses Clerk and preferences say Firebase Auth: **use Clerk**.

The only exception is when something is *missing entirely* — then preferences fill the gap.

| Detected | Preference | What to do |
|---|---|---|
| Prisma | Drizzle | Use Prisma |
| Clerk | Neon Auth / Better Auth | Use Clerk |
| Firebase Auth | Neon Auth / Better Auth | Extend Firebase Auth; do not migrate |
| Paddle | Stripe | Use Paddle (warn if you're less confident) |
| SendGrid | Resend | Use SendGrid |
| Vercel/Railway/Render | Vercel or Railway | Use whatever is already configured; never migrate deploy platforms as a side effect |
| Nothing | Drizzle | Install Drizzle |
| Nothing | Neon Auth / Better Auth | Install Neon Auth via Better Auth |

### 3. Mirror local conventions

When adding a new file, look at sibling files first (use `pattern-finder`). Match folder layout, file naming, import order, error handling, validation lib, async patterns. If conventions are inconsistent (vibe-coded), pick the **most-recent** file's pattern and note it.

### 4. Confirm before destructive changes

Always require explicit user confirmation before:

1. Schema migrations (any DB structure change)
2. Removing or replacing existing files
3. Adding/removing top-level dependencies

Show the planned change. Ask. Then execute.

### 5. Vibe-coded apps need triage, not big-bang refactors

When `codebase-classifier` returns `vibe-coded`:

- Don't propose sweeping rewrites. Identify the **smallest set of changes** for the requested feature.
- Surface inconsistencies as "I noticed X — out of scope for this skill, but worth fixing later." Don't try to fix.
- Prefer **additive** changes over modifying existing code.
- If a clean fix would touch many files, flag the tradeoff: "minimal (one new file) or thorough (refactor 4 existing) — which?"

## Adaptation Modes by Skill Type

### `add-*` skills (auth, payment, files, monitoring)

| Class | Strategy |
|---|---|
| Greenfield | Install from preferences. Full setup: env vars, deps, route, client wiring. |
| Wired | Detect existing integration. Extend it. Mirror file/folder pattern exactly. |
| Vibe-coded | Detect what's there. Identify gaps (missing webhook signature? missing rate limit? hardcoded secret?). Surface gaps. Make the requested addition. |

### `check-production` and `next-steps`

Read-only on any class. Output a structured report. Make no changes without explicit follow-up.

### `build-feature`

| Class | Strategy |
|---|---|
| Greenfield | Use the layered TDD workflow. |
| Wired | Mirror the project's layering. |
| Vibe-coded | Pick the most coherent existing layer and mirror that. Note inconsistency. |

### `setup-project`

Greenfield only. For wired/vibe-coded, redirect to `/next-steps` first.

### `migrate-from-vibe`

Vibe-coded only. Greenfield/wired don't need this skill.

### `deploy`

| Class | Strategy |
|---|---|
| Greenfield | Ask the user whether they want Vercel or Railway (no default), then walk through that platform phase by phase, with extensive browser-step instructions. |
| Wired/vibe-coded | Detect existing platform and adapt. If multiple deploy configs exist, ask which is real. |

### `triage`

Always works the same regardless of class. Writes the report to `.claude/bugs/`.

## When in Doubt

- **Don't overwrite.** Add a new file or extend in place additively.
- **Don't migrate.** If the user wants to migrate (Prisma → Drizzle), that's a separate explicit ask via `/refactor`.
- **Don't assume.** Detect first, ask second, write third.
- **Surface, don't fix.** Out-of-scope issues get noted in the skill's final report; the user decides whether to address them in a follow-up.
