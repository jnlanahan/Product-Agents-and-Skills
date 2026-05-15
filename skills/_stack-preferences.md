---
name: _stack-preferences
description: Reference data — the user's definitive SaaS stack for greenfield projects. Read by /setup-project and any /add-* skill that needs to know what to install when nothing is detected. Not invocable directly.
---

# User's Stack Preferences (Greenfield Defaults)

These are the **required defaults** for brand-new projects — not suggestions, not preferences. When working on an existing app that already uses something different, **follow the existing app's pattern** — never migrate as a side effect. See `_adaptation-playbook.md`.

## The Stack

| Layer | Choice | Notes |
|---|---|---|
| Language | **TypeScript** | Strict mode; catches bugs before runtime |
| Meta-framework | **Next.js (App Router)** | Frontend + secure backend in one codebase; serverless by default |
| UI components | **shadcn/ui + Tailwind CSS** | Copy-paste components into project; Tailwind for all styling |
| DB | **Neon Postgres** | Serverless Postgres; scales to zero; Git-like branching for safe prototyping |
| ORM | **Drizzle** | Type-safe queries; schema in `db/schema.ts`; `drizzle-zod` for validators |
| Auth | **Neon Auth (via Better Auth)** | User tables live inside your Neon Postgres DB; Google Sign-In built-in |
| Payments | **Stripe** | Checkout Sessions + Customer Portal; webhook signature mandatory |
| File storage | **AWS S3 + CloudFront CDN** | S3 holds assets; CloudFront delivers them securely and cheaply at scale |
| Env validation | **t3-env** | App crashes at startup if a required env var is missing — no silent failures |
| Deploy | **Vercel (Hobby Tier)** | Built for Next.js; auto-deploys on every GitHub push; free tier covers most projects |
| Version control | **GitHub** | Triggers Vercel deployments; links to Neon DB branches |
| Analytics | **PostHog** | User tracking, session replay, feature flags (optional; wire from day one on SaaS) |
| Errors | **Sentry (Free Tier)** | Captures exact error line and alerts automatically |
| AI | **Vercel AI SDK (`ai` package)** | Swap models (OpenAI/Anthropic/Google) with one syntax; only wire if AI features planned |
| Validation | **Zod** | Pair with `drizzle-zod`; validate at every API boundary |

## Greenfield Scaffold Order

Each wave is one commit.

1. **Framework + TypeScript + Tailwind + shadcn/ui** — `create-next-app`, strict tsconfig, Tailwind config, shadcn init
2. **Drizzle + Neon** — DB connection, schema with `users` table, first migration
3. **Neon Auth (via Better Auth)** — Better Auth config, Google Sign-In, session middleware, protected route pattern
4. **t3-env** — Validate all env vars at startup; generate `.env.example`
5. **Stripe** — Checkout endpoint, webhook handler with signature verification, Customer Portal
6. **AWS S3 + CloudFront** — Upload route, pre-signed URL pattern, CloudFront delivery, IAM policy
7. **Sentry + PostHog** — Error monitoring and product analytics wired into layout; PostHog is optional for personal projects
8. **AI SDK** — Only if AI features are planned; skip otherwise
9. **Security middleware** — Rate limiting, CORS, security headers (Next.js config or middleware.ts)
10. **Test infrastructure** — Vitest, one smoke test, `npm test` script
11. **CI** — `.github/workflows/test.yml`; typecheck + tests on every PR
12. **Health check + docs** — `/api/health` endpoint, complete `.env.example`, README quick-start, `CLAUDE.md` skills index

## Required Environment Variables (greenfield baseline)

```
# Database (Neon)
DATABASE_URL=

# Auth (Better Auth / Neon Auth)
BETTER_AUTH_SECRET=          # Long random string — generate with: openssl rand -base64 32
BETTER_AUTH_URL=             # Your app's base URL, e.g. http://localhost:3000
AUTH_GOOGLE_CLIENT_ID=       # From Google Cloud Console OAuth credentials
AUTH_GOOGLE_CLIENT_SECRET=   # From Google Cloud Console OAuth credentials

# Payments (Stripe)
STRIPE_SECRET_KEY=
STRIPE_WEBHOOK_SECRET=
NEXT_PUBLIC_STRIPE_PUBLISHABLE_KEY=

# File Storage (AWS S3 + CloudFront)
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=
AWS_S3_BUCKET=
CLOUDFRONT_DOMAIN=           # e.g. d1234abcd.cloudfront.net
CLOUDFRONT_KEY_PAIR_ID=      # From CloudFront key groups in AWS Console
CLOUDFRONT_PRIVATE_KEY=      # PEM private key for signing CloudFront URLs

# Monitoring
SENTRY_DSN=
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=           # For source map uploads in CI
NEXT_PUBLIC_POSTHOG_KEY=     # Optional
NEXT_PUBLIC_POSTHOG_HOST=    # Optional
```

## Why these choices (one-line each)

- **Neon Auth (Better Auth)** — Auth and user data live in the same Postgres DB; no separate Firebase project to manage
- **Neon + Drizzle** — Serverless Postgres with fully type-safe queries; Drizzle handles migrations automatically
- **AWS S3 + CloudFront** — S3 is the industry standard for object storage; CloudFront prevents public bucket exposure and cuts egress costs
- **Vercel** — Zero-config deployment for Next.js; every GitHub push is live in seconds
- **t3-env** — Missing env vars cause loud startup failures instead of silent runtime bugs
- **Sentry** — Captures the exact error line in production; free tier covers most projects
- **shadcn/ui** — Copy-paste components that live in your codebase; no dependency lock-in
