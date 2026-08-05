---
name: 0-Setup-Align-Stack
description: MUST BE USED when the user wants to migrate an existing project's integrations to their preferred stack — Neon Postgres + Drizzle, Neon Auth via Better Auth, Stripe, AWS S3 + CloudFront, Sentry, PostHog, Vercel AI SDK, Zod, and deploy on Vercel or Railway. Detects what's currently wired, shows a gap table (current vs. target), sequences migrations by risk, gets explicit approval per layer, then executes one wave per layer with one commit each. Never migrates live auth or active subscriptions without a documented migration plan, and never migrates between Vercel and Railway as a side effect.
when_to_use: "User says 'migrate to my preferred stack', 'convert to Neon', 'switch from Supabase', 'replace Clerk', 'off Firebase Auth', 'switch to Better Auth', 'replace Prisma with Drizzle'."
---

# /0-Setup-Align-Stack

You migrate an existing project's integrations to the user's preferred stack — deliberately and layer-by-layer. You detect what's currently wired, build a gap table, get the user's approval on each wave, then execute with one commit per wave. You never silently swap integrations that affect live data (user accounts, active subscriptions) — those always require an explicit migration plan.

This skill is the intentional counterpart to `_adaptation-playbook.md`'s "existing patterns win" rule. That rule governs skills that add features. This skill is invoked specifically to *change* existing patterns to match the preferred stack.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- **Assess everything before changing anything.** Complete the gap table (Step 2) before touching files.
- **One wave per integration layer, one commit per wave.** Rollback is cheap; big-bang swaps are not.
- **Auth hard stop:** if real production users exist, do NOT just swap auth providers — sessions break and accounts are lost. See Auth wave below.
- **Payments hard stop:** if active subscriptions exist, do NOT swap payment processors without a documented manual migration plan. See Payments wave below.
- **Verify between every wave.** `npm run build` must pass before the next wave starts.

## When to Use

- Project uses a different stack (Supabase, Firebase, Prisma, Clerk, PlanetScale, etc.) and the user explicitly wants to standardize on their preferred stack
- User is adopting a client project or fork and wants to bring it to their stack
- Project came off `/4-Build-Migrate-From-Vibe` with old integrations still wired; user now wants to convert them
- User is doing a deliberate tech stack modernization pass

## When NOT to Use

- Just adding a missing integration → use `/4-Build-Auth`, `/add-database`, `/4-Build-Payments`, etc.
- Rehabilitating vibe-coded mess without a specific stack migration goal → use `/0-Setup-Unvibe` first
- Brand-new project with no integrations → use `/0-Setup-Project`
- User explicitly says "keep the existing stack" — this skill is opt-in migration only

## Preferred Stack (migration target)

| Layer | Target |
|---|---|
| Database | **Neon Postgres** |
| ORM | **Drizzle** (`db/schema.ts` + `drizzle-zod`) |
| Auth | **Neon Auth via Better Auth** (Google Sign-In; user table in Neon) |
| Payments | **Stripe** (Checkout Sessions + Customer Portal) |
| File storage | **AWS S3 + CloudFront CDN** |
| Monitoring | **Sentry** (errors) + **PostHog** (analytics) |
| AI | **Vercel AI SDK** (`ai` package) |
| Validation | **Zod** |
| Deploy | **Vercel or Railway** (co-equal; ask the user — never migrate between them as a side effect) |

→ Full stack rationale: [skills/_stack-preferences.md](../_stack-preferences.md)

---

## Procedure

### Step 1: Detect current stack

Run in parallel:
- `stack-detector` — full STACK PROFILE for all layers
- `codebase-classifier` — `greenfield` / `wired` / `vibe-coded`

If `codebase-classifier` returns `vibe-coded`, surface a warning:

> This codebase is vibe-coded. Consider running `/0-Setup-Unvibe` first — migrating integrations on top of messy code doubles the risk. Want to run `/0-Setup-Unvibe` first, or proceed with `/0-Setup-Align-Stack` anyway?

Otherwise continue.

### Step 2: Build the gap table

Produce a table showing current vs. target for every layer. Mark each with:
- `aligned` — already on target; skip this wave
- `needs-migration` — active swap required
- `missing` — not wired at all; install fresh per the matching `/add-*` skill

Example:

| Layer | Current | Target | Status |
|---|---|---|---|
| Database | Supabase Postgres | Neon Postgres | `needs-migration` |
| ORM | Prisma | Drizzle | `needs-migration` |
| Auth | Clerk | Neon Auth / Better Auth | `needs-migration` |
| Payments | Lemon Squeezy | Stripe | `needs-migration` |
| File storage | Cloudinary | S3 + CloudFront | `needs-migration` |
| Monitoring | none | Sentry + PostHog | `missing` |
| AI | OpenAI SDK | Vercel AI SDK | `needs-migration` |
| Validation | Yup | Zod | `needs-migration` |
| Deploy | Railway | Vercel or Railway | `aligned` (Railway is an accepted target — don't migrate it) |

Show the table to the user. Ask:
> Which layers do you want to migrate? I'll sequence them lowest-risk first. You can defer any layer.

### Step 3: Sequence and plan

Present approved layers in this risk-ascending order:

1. **Validation** — no data, no external deps; safest first
2. **AI SDK** — same LLM calls, different wrapper; low blast radius
3. **Monitoring** — additive; old can stay in parallel temporarily
4. **ORM** — schema rewrite, no data loss if DB stays the same
5. **Database** — host migration; requires data export/import if data exists
6. **File storage** — URL rewriting required in the database
7. **Deploy** — DNS cutover; configuration-heavy but code-light
8. **Auth** — high risk if real users exist (hard stop)
9. **Payments** — highest risk if active subscriptions exist (hard stop)

Write `.claude/0-Setup-Align-Stack-plan.md`:

```markdown
# Align Stack Plan — <project-name> — <date>

## Gap summary
<paste gap table>

## Approved waves (in order)
- [ ] Wave 1: <layer> — <current> → <target>
- [ ] Wave 2: ...

## Deferred
- <layer>: <reason>

## Decisions log
(filled in as waves complete)
```

Say to the user:
> Plan written to `.claude/0-Setup-Align-Stack-plan.md`. I'll walk through it wave-by-wave; you approve each before I touch anything. Ready for Wave 1 (<layer>)?

### Step 4 onwards: Execute each wave

For every approved wave:

1. Confirm what will change and which env vars are needed
2. Install new package(s)
3. Rewrite config, schema, routes, client hooks
4. Uninstall old package(s)
5. Update `.env.example` with new vars; comment out old ones
6. Run `npm run build && npm test` (or project equivalent)
7. Commit: `"align-stack: <layer> → <target>"`
8. Update `.claude/0-Setup-Align-Stack-plan.md` checkboxes

---

## Per-Layer Procedures

### Validation: Yup / Joi / class-validator → Zod

1. `npm install zod`
2. Uninstall old validator package
3. Rewrite every schema: `z.object({ field: z.string() })` etc.
4. Replace `.validate()` / `validate(schema, data)` calls with `schema.parse(req.body)` or `schema.safeParse()`
5. Once Drizzle is in place, add `drizzle-zod` and derive schemas with `createInsertSchema(table)` / `createSelectSchema(table)`
6. Commit: `"align-stack: [old validator] → Zod"`

### AI SDK: Provider SDK → Vercel AI SDK

1. `npm install ai @ai-sdk/anthropic` (or `@ai-sdk/openai`, `@ai-sdk/google` — whichever the LLM)
2. Uninstall `openai`, `@anthropic-ai/sdk`, `langchain`, etc.
3. Replace completions / streams with `streamText`, `generateText`, `generateObject` from `ai`
4. Wire model: `import { anthropic } from '@ai-sdk/anthropic'` (keep the same API key env var)
5. Commit: `"align-stack: [old AI SDK] → Vercel AI SDK"`

→ For complex AI integrations run `/4-Build-AI` for the full guided setup.

### Monitoring: Any → Sentry + PostHog

1. `npm install @sentry/nextjs` (or `@sentry/node` for non-Next projects)
2. Wire `Sentry.init()` in the instrumentation file; add `SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `NEXT_PUBLIC_SENTRY_DSN` to `.env.example`
3. Remove old error monitoring (LogRocket, Rollbar, Bugsnag)
4. `npm install posthog-js posthog-node`
5. Wire PostHog provider in `layout.tsx`; add `NEXT_PUBLIC_POSTHOG_KEY`, `NEXT_PUBLIC_POSTHOG_HOST` to `.env.example`
6. Remove old analytics (Mixpanel, Amplitude, Segment, GA4)
7. Commit: `"align-stack: monitoring → Sentry + PostHog"`

→ See [skills/4-Build-Monitoring/SKILL.md](../4-Build-Monitoring/SKILL.md) for full wiring detail.

### ORM: Prisma / TypeORM / raw SQL → Drizzle

→ Full procedure: [references/database-migration.md](references/database-migration.md)

1. `npm install drizzle-orm @neondatabase/serverless drizzle-kit drizzle-zod`
2. Translate source schema → `db/schema.ts` (Drizzle column types)
3. Run `npx drizzle-kit generate`; review SQL output with user before applying
4. Rewrite every query from Prisma/TypeORM syntax to Drizzle equivalent
5. Remove `@prisma/client`, `prisma`, `typeorm`, etc.; delete `prisma/` folder
6. Add `db:generate`, `db:migrate`, `db:push`, `db:studio` scripts to `package.json`
7. Run build; fix TypeScript errors
8. Commit: `"align-stack: [old ORM] → Drizzle"`

### Database: Supabase / PlanetScale / Vercel Postgres / Railway Postgres → Neon

→ Full procedure: [references/database-migration.md](references/database-migration.md)

**Before starting, ask:** "Does this database have real production data?"
- **No data / dev environment:** switch `DATABASE_URL` to Neon, run migrations, done.
- **Data exists:** walk through the export/import steps in the reference doc; do not drop the source DB until row counts match in Neon.

1. User creates a Neon project in the Neon Console; provide the connection string
2. Add `DATABASE_URL` (Neon connection string + `?sslmode=require`) to `.env.local`
3. `npx drizzle-kit migrate` to apply schema to Neon
4. If data exists: `pg_dump` from source → `pg_restore` to Neon; verify row counts
5. Update all env references; remove old DB env vars from `.env.example`
6. Smoke test: at least one DB read and one DB write
7. Commit: `"align-stack: database → Neon Postgres"`

### File Storage: Cloudinary / Firebase Storage / Uploadthing / Supabase Storage → S3 + CloudFront

1. User creates S3 bucket + CloudFront distribution; provide the IAM policy template (least-privilege: `s3:PutObject`, `s3:GetObject`, `s3:DeleteObject` on the bucket ARN only)
2. Add `AWS_ACCESS_KEY_ID`, `AWS_SECRET_ACCESS_KEY`, `AWS_REGION`, `AWS_S3_BUCKET`, `CLOUDFRONT_DOMAIN` to `.env.example`
3. `npm install @aws-sdk/client-s3 @aws-sdk/s3-request-presigner`
4. Rewrite upload routes: generate pre-signed PUT URLs → client uploads directly to S3
5. Rewrite file-serving: signed CloudFront URLs (private assets) or plain CloudFront domain (public assets)
6. **URL migration in DB:** any file URLs stored in the DB (old CDN domain) must be updated — show the `UPDATE` SQL and ask for approval before running
7. Uninstall old storage SDK
8. Commit: `"align-stack: file storage → S3 + CloudFront"`

→ See [skills/4-Build-File-Storage/SKILL.md](../4-Build-File-Storage/SKILL.md) for full wiring detail.

### Deploy: Render / Heroku / fly.io → Vercel or Railway

**Vercel and Railway are both accepted targets — if the project is already on either one, it's `aligned`; do not migrate between them.** Only migrate when the current platform is something else (Render, Heroku, fly.io, etc.). Ask the user which target they want — there is no default.

1. Confirm the framework. Vercel is ideal for Next.js (serverless); Railway suits long-running servers, background workers, or websockets. For Express/Vite, ask whether they want serverless functions (Vercel) or a single container host (Railway).
2. Walk through [platform setup steps](../6-Deploy/references/platform-setup-steps.md) for the chosen platform
3. Wire all env vars on the target platform (Vercel dashboard, or Railway → service → Variables — import from `.env.example`)
4. Connect GitHub repo; confirm auto-deploy on push is active
5. Test: push a commit, confirm the build completes and the preview/live URL works
6. DNS cutover: provide A/CNAME values for the chosen platform; user does the registrar change
7. Decommission old platform only after DNS has propagated and the app is live on the new domain for 24 h
8. Commit: `"align-stack: wired <Vercel|Railway> deployment"` (usually config-only; no source changes)

→ See [skills/6-Deploy/SKILL.md](../6-Deploy/SKILL.md) for the full deploy guide.

### Auth: Any → Neon Auth via Better Auth

**Hard stop — answer this before writing a single line of code:**

> Are there real users with live accounts in this auth system?
>
> - **No users / dev only:** proceed with the standard swap below.
> - **Real users exist:** sessions will break and accounts will be inaccessible if you simply swap providers. Choose a strategy:
>   - **Option A — parallel run:** wire Better Auth alongside the old provider; migrate users in batches by email match; deprecate old provider after all accounts are active in Better Auth.
>   - **Option B — hard cut with re-registration:** appropriate only for very small user bases (< 50 users) who agree to re-auth. Announce the change in advance.
>   - **Option C — defer:** keep existing auth provider as-is and revisit migration when you have more runway.
>
> Surface this finding and **stop until the user makes an explicit choice.**

Standard swap (no real users):

1. `npm install better-auth`
2. Create `auth.ts` at project root with Neon Postgres adapter and Google OAuth
3. Add `BETTER_AUTH_SECRET`, `BETTER_AUTH_URL`, `AUTH_GOOGLE_CLIENT_ID`, `AUTH_GOOGLE_CLIENT_SECRET` to `.env.example`
4. Add API route: `app/api/auth/[...all]/route.ts` → `toNextJsHandler(auth.handler)`
5. Replace old auth calls: `getServerSession()` / `currentUser()` / `useUser()` → Better Auth session helpers
6. Move sign-in / sign-up pages to Better Auth's redirect flow
7. Uninstall old auth packages (Clerk, `next-auth`, `@supabase/auth-helpers-nextjs`, `firebase/auth`, etc.)
8. Commit: `"align-stack: auth → Neon Auth via Better Auth"`

→ Detailed per-provider procedure: [references/auth-migration.md](references/auth-migration.md)
→ Full setup guide: [skills/4-Build-Auth/SKILL.md](../4-Build-Auth/SKILL.md)

### Payments: Lemon Squeezy / Paddle / Braintree → Stripe

**Hard stop — answer this before writing a single line of code:**

> Are there active paying subscribers in this payment system?
>
> - **No live subscriptions / pre-launch:** proceed with the standard swap below.
> - **Active subscriptions exist:** payment processor migrations with live subscriptions require manual steps that cannot be automated safely:
>   1. Export subscriber list from old processor
>   2. Create matching Stripe Customer + Subscription records via Stripe API or CSV import
>   3. Coordinate a parallel billing window so customers are not double-charged
>   4. Cancel old subscriptions only after Stripe ones are confirmed active
>   5. Notify affected customers of the change
>
>   This is a business-critical manual operation. Provide the checklist; require the user to acknowledge each step; do NOT code the migration until the business steps are confirmed in place.

Standard swap (no live subscriptions):

1. `npm install stripe`
2. Add `STRIPE_SECRET_KEY`, `STRIPE_WEBHOOK_SECRET`, `NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY` to `.env.example`
3. Create `lib/stripe.ts`: `export const stripe = new Stripe(process.env.STRIPE_SECRET_KEY!)`
4. Create Checkout Session endpoint (`/api/checkout`) and Customer Portal endpoint (`/api/portal`)
5. Create webhook handler (`/api/webhooks/stripe`) with `stripe.webhooks.constructEvent()` signature verification — this is mandatory, not optional
6. Remove old processor routes, SDK calls, and webhook handlers
7. Uninstall old payment packages
8. Commit: `"align-stack: payments → Stripe"`

→ Full setup guide: [skills/4-Build-Payments/SKILL.md](../4-Build-Payments/SKILL.md)

---

## Hand Off

After all approved waves complete:

> Stack alignment complete.
> - Migrated: <list>
> - Deferred: <list>
> - Commits: <count>
>
> Recommended next:
> - `/5-Validate-Production-Readiness` — full audit now that the stack is new
> - `/0-Next` — see current project state
> - For deferred layers, rerun `/0-Setup-Align-Stack` when ready

---

## Rules

- **Read-only until Step 4.** Steps 1–3 write only to `.claude/0-Setup-Align-Stack-plan.md`.
- **One wave per layer, one commit per wave.** Never bundle two layers in one commit.
- **Verify after every wave.** `npm run build` must pass; blocked build = blocked wave.
- **Auth hard stop if real users exist.** No exceptions.
- **Payments hard stop if active subscriptions exist.** No exceptions.
- **Never delete old env vars from running system until new integration is verified.** Keep old vars in `.env.local` until smoke tests pass.
- **Don't drift into feature work.** Scope is wiring the integration, not improving the feature.
- **`_adaptation-playbook.md` "existing patterns win" does not apply here.** This skill is the explicit user-requested exception to that rule.

## If Something Goes Wrong

- **Build breaks after a wave** — `git revert HEAD`, diagnose, re-attempt with narrower scope
- **Auth sessions not persisting after swap** — confirm `BETTER_AUTH_URL` exactly matches the app's running origin (including `http://` vs `https://` and no trailing slash)
- **Database data loss during migration** — restore from source DB (which was not dropped); add missing rows; re-run migration
- **Stripe webhook signature fails in dev** — use `stripe listen --print-secret` for the dev webhook secret; the dashboard webhook secret is for production only
- **Old integration still referenced after swap** — run `grep -r "[old-package-name]" .` to find stale imports; clean up before closing the wave
