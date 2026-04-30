---
name: _stack-preferences
description: Reference data — the user's preferred SaaS stack for greenfield projects. Read by /setup-project and any /add-* skill that needs to know what to install when nothing is detected. Not invocable directly.
---

# User's Stack Preferences (Greenfield Defaults)

These are the defaults for **brand-new** projects. When working on an existing app that already uses something different, **follow the existing app's pattern** — never migrate as a side effect. See `_adaptation-playbook.md`.

## The Stack

| Layer | Choice | Notes |
|---|---|---|
| Auth | **Firebase Auth** | Bearer ID tokens to API; admin SDK on server |
| Payments | **Stripe** | Checkout Sessions + Customer Portal; webhook signature mandatory |
| DB | **Neon Postgres** | Serverless driver |
| ORM | **Drizzle** | Schema in `shared/schema.ts` or `db/schema.ts`; `drizzle-zod` for validators |
| File storage | **Firebase Storage** | Security rules in `firebase-storage.rules`; ownership checks server-side |
| Frontend | **Detect first** | Next.js App Router preferred for new; respect existing if present |
| Backend | **Detect first** | Next.js Server Actions/API routes if Next.js; else Express |
| Deploy | **Render or Railway** | Either supports Express and Next.js |
| Analytics | **PostHog** | Required — user tracking, session replay, feature flags |
| Errors | **Sentry** | Required — pairs with PostHog (Sentry for stack traces, PostHog for product) |
| AI | **Vercel AI SDK v5** | `ai` + `@ai-sdk/<provider>`; supports OpenAI/Anthropic/Google |
| Validation | **Zod** | Pair with `drizzle-zod`; validate at every API boundary |
| Security | **Helmet + express-rate-limit + cors** (Express) or Next.js equivalents | |

## Greenfield Scaffold Order

1. Framework + TypeScript
2. Drizzle + Neon
3. Firebase Auth
4. Stripe
5. Firebase Storage
6. PostHog + Sentry
7. AI SDK (only if AI features requested)
8. Security middleware
9. Test infrastructure (Vitest or Jest)
10. CI (GitHub Actions: typecheck + tests on PR)
11. Health check + `.env.example` + CLAUDE.md with skills index

## Required Environment Variables (greenfield baseline)

```
DATABASE_URL=
FIREBASE_SERVICE_ACCOUNT=                # JSON, server-side only
NEXT_PUBLIC_FIREBASE_*=                   # Client config
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=
SENTRY_DSN=
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=                        # For source map uploads in CI
```

(For non-Next.js stacks, replace `NEXT_PUBLIC_` with `VITE_` or appropriate.)

## Why these choices (one-line each)

- **Firebase Auth** — works well alongside Firebase Storage as a unit
- **Neon + Drizzle** — clean Postgres + best-in-class TS ORM, decoupled from auth
- **Stripe** — mature webhook + Customer Portal story
- **Sentry + PostHog (both)** — they don't overlap; one for errors, one for product analytics
- **Render/Railway** — work with both Express and Next.js
