---
name: 0-Setup-Project
description: MUST BE USED when starting a brand-new SaaS project from empty or fresh-scaffold state. Walks the user through third-party account setup, scaffolds the preferred stack in disciplined waves (one commit per wave), generates a CLAUDE.md with a skills index, and seeds git/GitHub conventions. Use `--personal` flag for lighter personal tools (no Stripe, optional auth, SQLite). NOT for existing projects — for those, run `/0-Next-steps` first.
when_to_use: "User says 'start a new project', 'new SaaS app', 'scaffold this from scratch', 'I'm building a new app'."
disable-model-invocation: true
---

# /0-Setup-Project

You set up a new project from near-empty state using the user's definitive stack from `_stack-preferences.md`. Discipline matters here: each layer goes in as a separate commit, in the right order, with verification before moving on.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Only run on an empty directory or fresh scaffold — this skill scaffolds opinionated structure and will overwrite conflicting files.
- For existing projects, run `/0-Next-steps` first; do NOT run `/0-Setup-Project` on a project that already has real code.
- Verify the target directory before starting (`ls`) and confirm it is empty or contains only a README or basic scaffold.

## When to Use

- Empty directory or `npm init -y` output
- Fresh `create-next-app` scaffold with no real features
- User explicitly says "start a new SaaS project" or "start a personal tool"

## When NOT to Use

- Project has source files beyond a scaffold → run `/0-Next-steps` first to assess
- Project has an existing auth/payment/db setup → use the targeted `/add-*` skill instead
- Project came from a vibe-coding tool (Replit/V0/Lovable/Bolt) → run `/4-Build-Migrate-From-Vibe` first

## `--personal` Mode (lighter stack for personal tools)

When invoked as `/0-Setup-Project --personal`, use a reduced scope:

| Layer | SaaS default | Personal (`--personal`) |
|---|---|---|
| DB | Neon Postgres + Drizzle | SQLite + Drizzle (local-first, no cloud account needed) |
| Auth | Neon Auth via Better Auth | Optional — ask if needed |
| Payments | Stripe | Skip |
| File storage | AWS S3 + CloudFront | Skip |
| Monitoring | Sentry + PostHog | Sentry only (optional) |
| Deploy | Vercel | Vercel (same; free hobby tier works) |

Everything else (test infra, CI, CLAUDE.md, security middleware) still applies. Use `--personal` for: internal dashboards, scripts with a UI, personal productivity tools, side projects without a business model.

Skip the "External account setup" step for omitted services. Ask only about the accounts that are actually needed.

## Procedure

### Step 1: Verify greenfield

Run `stack-detector` and `codebase-classifier` in parallel. If classification is **not** `greenfield`:

> Stop. Tell the user: "This project is `<wired|vibe-coded>`. `/0-Setup-Project` is greenfield only. Run `/0-Next-steps` to understand what's there, then use targeted `/add-*` skills."

### Step 2: Confirm framework

Read `_stack-preferences.md`. Confirm with the user:

> We'll use Next.js App Router (TypeScript, Tailwind CSS, shadcn/ui) — the definitive stack for new SaaS projects. This gives you a frontend and a secure backend in one codebase.
>
> Is this correct, or do you have a different framework in mind?

### Step 3: Confirm scope

Ask which integrations to wire now (defaults all on):

- [x] Drizzle + Neon (almost always needed)
- [x] Neon Auth via Better Auth (almost always needed)
- [x] t3-env (always — validates env vars at startup)
- [x] Stripe (defer if pre-revenue)
- [x] AWS S3 + CloudFront (defer if no file uploads needed)
- [x] Sentry + PostHog (wire from day one)
- [ ] AI SDK (only if AI features planned)

User can opt out of any. Explain trade-offs if they opt out of t3-env or Sentry.

### Step 4: External account setup (USER does these in browser FIRST)

Before installing any code, the user needs accounts and credentials. Tell them, verbatim:

> **You need these accounts before I install anything. Open each, sign up, and grab the values listed. Take your time — I'll wait.**
>
> 1. **Neon** (neon.tech) — sign up free → create project → copy `DATABASE_URL` from Connection Details
> 2. **Google Cloud Console** (console.cloud.google.com) — create a project → APIs & Services → Credentials → Create OAuth 2.0 Client ID (Web application) → add redirect URI `http://localhost:3000/api/auth/callback/google` → copy Client ID and Client Secret
> 3. **Stripe** (dashboard.stripe.com/register) — sign up → leave in TEST mode → Developers → API keys → copy `pk_test_*` and `sk_test_*`
> 4. **AWS** (aws.amazon.com) — sign up (free tier) → create S3 bucket (block all public access) → create IAM user with S3 access → copy Access Key ID and Secret Access Key → create CloudFront distribution pointing at your bucket → copy the distribution domain name
> 5. **Sentry** (sentry.io) — sign up → create project (Next.js) → copy DSN → Settings → Auth Tokens → create with `project:releases` scope
> 6. **PostHog** (posthog.com) — sign up → copy Project API Key and Host URL (optional but recommended)
> 7. **GitHub** (github.com) — sign up if needed; you'll link a repo at the end of setup
> 8. **Vercel** (vercel.com) — sign up with GitHub → you'll link the repo after git init
>
> Reply with all the values, or "I have them ready — let's go."

Wait for the user.

### Step 5: Show the install plan

Show the full list of dependencies, files to create, files to modify, env vars needed. Get explicit approval before any `npm install`.

### Step 6: Execute in waves (one commit per wave)

Each wave is a single commit — easier to roll back, easier to debug. After each wave: `npm run build`. Fix before moving on.

**Wave 1: Framework + TypeScript + UI**
- `npx create-next-app@latest` with TypeScript, Tailwind, App Router
- Initialize shadcn/ui: `npx shadcn@latest init`
- Ensure tsconfig strict mode
- Commit: "scaffold framework + tailwind + shadcn"

**Wave 2: DB + ORM**
- Install Drizzle + Neon serverless driver
- Drizzle config, `db/schema.ts` with `users` table, first migration
- Commit: "wire drizzle + neon"

**Wave 3: Auth (Neon Auth via Better Auth)**
- Install `better-auth`
- Create `lib/auth.ts`, `app/api/auth/[...all]/route.ts`, `lib/auth-client.ts`
- Session middleware in `middleware.ts`, sign-in page
- Google Sign-In configured
- Commit: "wire neon auth via better-auth"

**Wave 4: Env validation (t3-env)**
- Install `@t3-oss/env-nextjs`
- Create `src/env.ts` validating all env vars added so far
- Generate complete `.env.example`
- Commit: "add t3-env validation"

**Wave 5: Payments**
- Stripe client, Checkout Session endpoint, webhook handler with signature verification, Customer Portal endpoint
- Commit: "wire stripe checkout + webhook"

**Wave 6: File storage (AWS S3 + CloudFront)**
- Install `@aws-sdk/client-s3`, `@aws-sdk/s3-request-presigner`, `@aws-sdk/cloudfront-signer`
- Pre-signed upload route, CloudFront delivery helper, file metadata table in DB
- Commit: "wire aws s3 + cloudfront"

**Wave 7: Monitoring**
- Sentry: install, wire into `instrumentation.ts`, Next.js config, error boundaries
- PostHog: install, wire into layout (client-side)
- Commit: "wire sentry + posthog"

**Wave 8: AI SDK** (only if requested)
- Install `ai` + `@ai-sdk/<provider>`
- Commit: "wire vercel ai sdk"

**Wave 9: Security middleware**
- Rate limiting, CORS, security headers in `next.config.ts` or `middleware.ts`
- Commit: "add security middleware"

**Wave 10: Test infrastructure**
- Install Vitest, add `npm test` and `npm run test:watch` scripts
- Add one passing smoke test (e.g., `lib/utils.test.ts`)
- Commit: "add test infrastructure"

**Wave 11: CI**
- `.github/workflows/test.yml` — typecheck + tests on every PR
- Commit: "add ci workflow"

**Wave 12: Health check + docs**
- `/api/health` endpoint returning `{ status: 'ok' }` and app version
- Complete `.env.example` (all vars listed, no real values)
- README with quick-start instructions
- **Generate `CLAUDE.md` with the skills index** (see Step 7)
- Commit: "add health check, env example, readme, and claude.md"

**Wave 13: Deploy to Vercel**
- Confirm `.gitignore` covers `.env*`, `node_modules`, build outputs, `.claude/` (except committed planning docs)
- `git init` if not already done; push to GitHub
- Link GitHub repo to Vercel: `vercel link` (or walk through Vercel dashboard)
- Add all env vars to Vercel: Project Settings → Environment Variables
- Deploy: `vercel --prod` (or let Vercel auto-deploy from the GitHub push)
- Confirm the live URL works and `/api/health` returns 200
- Commit: (no new code; just push to remote)

After each wave: `npm run build && npm run typecheck`. If broken, fix before next wave.

### Step 7: Generate CLAUDE.md with skills index

→ See [claude-md-template.md](references/claude-md-template.md) for the exact `CLAUDE.md` structure to write at the repo root. Fill in the `<...>` placeholders with actual project values.

### Step 8: End-to-end verification

- [ ] `npm run dev` boots without errors
- [ ] `/api/health` returns 200
- [ ] Sign up + sign in flow works (Google Sign-In)
- [ ] Protected page redirects unauthenticated users to sign-in
- [ ] Stripe Checkout test card succeeds (4242 4242 4242 4242)
- [ ] `stripe listen --forward-to localhost:3000/api/webhooks/stripe` shows webhook events delivered
- [ ] File upload works; file appears in S3 bucket; CloudFront URL serves the file
- [ ] Sentry test exception appears in Sentry dashboard
- [ ] PostHog event appears in PostHog dashboard (if wired)
- [ ] CI passes on first PR (typecheck + tests green)
- [ ] Vercel preview URL works on a test PR
- [ ] `CLAUDE.md` exists at repo root and includes the Available Skills section

### Step 9: Hand off

> Project is ready for development. Next steps:
> - Run `/2-Define-PRD` to produce a PRD for your first feature
> - Then `/2-Define-Plan` to turn it into vertical slices
> - Then `/4-Build-Feature` to implement
> - Run `/0-Next-steps` anytime to see project state
> - When ready to ship: `/5-Validate-Production-Readiness` then `/6-Deploy`

## Rules

- **External accounts before code.** Don't install anything until the user has the credentials in hand.
- **One wave per commit.** Easier review, easier rollback.
- **Verify after each wave.** Don't proceed if `npm run build` is broken.
- **Don't write app features.** This skill wires infrastructure. The first user-facing feature is `/4-Build-Feature`.
- **Don't run `db:push` against a production database.** Local dev only. Production uses `db:migrate`.
- **Always generate `CLAUDE.md` with the skills index.** Without it, the agent can't suggest commands the user has forgotten.

## If Something Goes Wrong

- **Package install fails** — check Node version (18+); delete `node_modules` and lockfile, then retry.
- **First migration fails** — confirm the DATABASE_URL is correct and t3-env passes startup validation.
- **Better Auth session not persisting** — confirm `BETTER_AUTH_SECRET` is set and `BETTER_AUTH_URL` matches the running URL exactly.
- **Google OAuth redirect mismatch** — the redirect URI in Google Cloud Console must exactly match `BETTER_AUTH_URL + /api/auth/callback/google`.
- **Vercel deploy fails** — check that all env vars are set in Vercel Project Settings; missing env vars cause t3-env to throw at startup.
- **S3 upload fails with CORS error** — add a CORS configuration to the S3 bucket allowing PUT from your app domain.
- **CloudFront returns 403** — make sure the bucket policy was updated to grant CloudFront OAC access (AWS shows this policy during distribution setup).
