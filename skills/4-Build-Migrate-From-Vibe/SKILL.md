---
name: 4-Build-Migrate-From-Vibe
description: MUST BE USED when the user wants to move a project off a vibe-coding platform (Replit, V0, Lovable, Bolt, Cursor-only, ChatGPT-generated) onto a real local stack. Detects the source platform from file markers, maps env vars and integrations, extracts the working app, and rewires it onto the user's preferred stack. Preserves working features; flags inconsistencies as out-of-scope rather than fixing them.
when_to_use: "User says 'move off Replit', 'migrate from Lovable', 'extract from v0', 'get this off Bolt', 'move to a real local codebase'."
disable-model-invocation: true
---

# /4-Build-Migrate-From-Vibe

You move a project off a vibe-coding platform (Replit, V0, Lovable, Bolt) onto a real local stack the user can develop, test, and deploy normally. Preserve what works. Flag inconsistencies; don't fix them inline.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Complete the full inventory (Step 2) before touching any files — do not rename, move, or delete anything until the working feature list is confirmed.
- Never delete source platform files until the migrated version passes smoke tests; keep originals as a reference.
- Flag every inconsistency discovered during migration as out-of-scope rather than silently fixing it — surprises here cause regressions.

## When to Use

- User has a working-but-fragile app from Replit, V0, Lovable, Bolt, or similar
- User has copy-pasted ChatGPT/Claude generations into a codebase that doesn't have a clear stack
- User wants to deploy to Render/Railway/Vercel but the current setup is platform-locked

## When NOT to Use

- Brand new project → use `/0-Setup-Project`
- Already on a real local stack but messy → use `/2-Define-Refactor` or `/5-Validate-Production-Readiness`
- Production-deployed but on the source platform's hosting → bigger conversation; ask user first

## Procedure

### Step 1: Detect source platform

Look for file markers and conventions:

| Marker | Platform |
|---|---|
| `.replit`, `replit.nix` | **Replit** |
| `v0-user-next.config.js`, comments referencing v0.dev | **V0 (Vercel)** |
| `lovable.dev` references in README, `.lovable/` directory | **Lovable** |
| `bolt.new` or `stackblitz.com` references, `.stackblitz/` | **Bolt / StackBlitz** |
| Plain pasted Next.js + Tailwind + Supabase with no clear local dev story | **ChatGPT/Claude pasted** |
| Mix of inconsistent patterns, unfinished code, no tests | **Vibe-coded (generic)** |

If multiple markers, ask the user which platform was the actual source. Some apps move through several.

### Step 2: Inventory what exists

Read the current state:

- What does the app do? (run it locally if possible, or read the README/main page)
- What integrations are wired? (auth, db, payments, storage, AI)
- What env vars are referenced? (grep for `process.env.`)
- What dependencies are in `package.json` that look incomplete or unused?
- Are there tests? (probably not — that's normal for vibe-coded)
- Is there a CLAUDE.md or AGENTS.md? (probably not)

Produce a concise summary for the user: "Here's what I found." Wait for confirmation before touching anything.

### Step 3: Determine target stack

Read `_stack-preferences.md`. By default the target is:

- Next.js App Router (preferred) or React+Vite+Express (ask user)
- Drizzle + Neon Postgres
- Neon Auth via Better Auth (or whatever the source already had wired — don't migrate auth providers mid-port)
- Stripe (or whatever was wired)
- AWS S3 + CloudFront (or whatever storage was wired)
- Sentry + PostHog
- Deploy to Vercel

Confirm with user. If something different was wired (Supabase Auth, Vercel Postgres, etc.), keep it — don't migrate. Migrating an integration mid-port doubles the risk.

### Step 4: Plan the port

Show the user a plan with these sections:

**Things to preserve as-is**
- Working pages and components (copy verbatim where possible)
- Working API routes (copy with minor framework adjustments)
- Existing schema (translate to target ORM)
- Existing env var names (so old keys still work)

**Things to add (missing from source platform)**
- Local dev story (`npm run dev`, hot reload, no platform required)
- Real environment variable file (`.env.example` + `.env.local`)
- Dependency cleanup (remove platform-specific deps; add target-stack deps)
- Migration system (Drizzle migrations vs whatever was there)
- Test infrastructure (one passing smoke test so the suite isn't empty)
- CLAUDE.md with the skills index

**Things flagged as out-of-scope**
- Inconsistent code patterns (different files using different styles)
- Insecure-looking code (hardcoded admin emails, missing auth checks, etc.)
- Missing error handling
- TODOs and `// FIXME` comments

We'll list these in `OUT_OF_SCOPE.md` for the user to address with `/2-Define-Refactor` or `/5-Validate-Triage` after the port lands.

Get approval before executing.

### Step 5: Execute in waves (one commit per wave)

Mirror `/0-Setup-Project`'s wave discipline. One commit per wave, verify between each.

**Wave 1: Establish the new stack skeleton**
- Initialize the target framework (Next.js or Vite+Express)
- Set up tsconfig (strict), package.json scripts
- Confirm `npm run dev` boots a blank page
- Commit: "scaffold target stack"

**Wave 2: Port environment**
- Create `.env.example` with all referenced env vars from source
- Document which need new values vs which carry over
- Commit: "port env config"

**Wave 3: Port database (if any)**
- Translate source schema to Drizzle (or detected ORM)
- Run `db:generate`; review SQL with user
- Run `db:migrate`
- If source had data, walk user through export/import
- Commit: "port database schema"

**Wave 4: Port auth (if any)**
- Mirror what was wired in source — don't migrate provider
- Set up server-side token verification middleware
- Move sign-in / sign-up pages
- Commit: "port auth"

**Wave 5: Port routes / API**
- Move each route, adjusting only the framework wrapper
- Keep auth checks, validation, business logic intact
- Commit per logical group: "port <feature> routes"

**Wave 6: Port pages / components**
- Move pages and components as-is where possible
- Adjust only imports and framework primitives (e.g., `useNavigate` → `useRouter`)
- Commit per page: "port <page> UI"

**Wave 7: Add what was missing**
- Test infrastructure (Vitest or Jest)
- Sentry + PostHog (per `/4-Build-Monitoring`)
- Security middleware
- CI workflow
- CLAUDE.md
- Commit each: "add <thing>"

**Wave 8: Verify end-to-end**
- All routes respond correctly
- Sign-in → protected route → sign-out flow works
- Any DB writes/reads work
- Tests pass

**Wave 9: Write `OUT_OF_SCOPE.md`**
- List everything flagged as out-of-scope during Step 4
- Each item has: what it is, why it was deferred, what skill to run to address it
- Commit: "document out-of-scope items from migration"

### Step 6: Hand off

Tell the user:

> Migration complete. Run:
> - `/0-Next-steps` — see what stage the project is at now and what's next
> - `/5-Validate-Production-Readiness` — full audit (vibe-coded apps usually have hidden issues)
> - Address each `OUT_OF_SCOPE.md` item one at a time using `/2-Define-Refactor` or `/5-Validate-Triage`
> - When ready, `/6-Deploy` will walk you through production deployment

## Source-Platform Specific Gotchas

### Replit

- `process.env.REPLIT_*` won't exist outside Replit. Look for code that depends on it and rewire to standard env vars or remove.
- Replit DB (key-value) won't translate cleanly. Likely need to migrate to Postgres.
- `replit.nix` and `.replit` files are useless outside Replit; delete them.
- "Always-on" / Replit Deploy → replace with Vercel; this is a deploy-skill problem after the port.

### V0 (Vercel)

- V0 outputs Next.js + Tailwind. Mostly portable as-is.
- Watch for Vercel-specific deps (`@vercel/postgres`, `@vercel/kv`) — translate to Neon/Drizzle and Upstash.
- V0 components often use shadcn/ui — copy `components/ui/*` as-is.

### Lovable

- Generates Vite + React + Supabase by default.
- Supabase Auth + Postgres can stay wired if you want; otherwise this is a migration job for after the port.
- Lovable's project structure is fairly standard; usually the issue is missing tests, CI, monitoring.

### Bolt / StackBlitz

- Often outputs runnable code that doesn't actually have a `package.json` matching the imports.
- First job: get `npm install` to succeed without errors.

### ChatGPT/Claude pasted

- The most variable. Treat as a fresh `/0-Setup-Project` with the existing files as input.
- Inconsistent file patterns are the norm — don't try to harmonize during the port.

## Rules

- **Don't migrate integrations during the port.** Same auth, same DB, same payments. The port is risky enough.
- **One commit per wave.** Easy revert at any step.
- **Verify between waves.** `npm run build && npm run dev` after each.
- **Flag, don't fix.** Inconsistencies, security issues, missing patterns → `OUT_OF_SCOPE.md`. Address them later with the right skill.
- **Preserve working features bit-for-bit.** A working page that works the same after the port is a win. A "improved" page that breaks subtly is a loss.
- **After the port, run `/0-Next-steps` and `/5-Validate-Production-Readiness`** — don't claim done until those pass.

## If Something Goes Wrong

- **Source platform files are unreadable or missing** — ask the user to export the project as a ZIP from the platform before running this skill; do not proceed without the source files.
- **Env vars are undocumented** — check the platform's "Settings" or "Secrets" panel; scan the source code for `process.env.*` or `import.meta.env.*` references and list them as unknowns.
- **A working feature breaks after migration** — restore the original source files and compare the broken version against the original; do not patch blindly.
- **Database schema is inconsistent** — flag all inconsistencies in a "Known Issues" section of the migration notes; do not silently fix them as they may have been intentional.