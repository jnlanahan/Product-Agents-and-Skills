---
title: "Sentry and PostHog Code Patterns"
skill: "4-Build-Monitoring"
---

# Sentry and PostHog Code Patterns

## Sentry Installation

### Next.js (via wizard — preferred)

```bash
npx @sentry/wizard@latest -i nextjs
```

The wizard:
- Installs `@sentry/nextjs`
- Creates `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`
- Wraps `next.config.ts` with `withSentryConfig`
- Creates `instrumentation.ts` if missing
- Asks for DSN and auth token interactively

### Other frameworks (manual)

```bash
npm install @sentry/node @sentry/react   # Express + React
# or
npm install @sentry/react                # Vite-only
```

Then add `Sentry.init({ dsn: process.env.SENTRY_DSN, tracesSampleRate: 0.1 })` early in the entrypoint.

---

## PostHog Installation

```bash
npm install posthog-js posthog-node
```

Create `lib/posthog-client.ts` (client) and `lib/posthog-server.ts` (server). Mirror the project's existing lib/* style. The client wraps `<PostHogProvider client={posthog}>` around your root layout. The server uses `posthog-node` for capture-then-shutdown in serverless.

Use `pattern-finder` to find the project's provider-mount pattern (e.g., is there already a providers wrapper in `app/layout.tsx`?). Mount PostHog there.

---

## User Identification (wire after sign-in)

```typescript
import * as Sentry from '@sentry/nextjs';
import posthog from 'posthog-js';

// After successful sign-in (client-side):
Sentry.setUser({ id: user.uid, email: user.email });
posthog.identify(user.uid, { email: user.email, plan: user.plan });

// On sign-out:
Sentry.setUser(null);
posthog.reset();
```

---

## Env Vars

Add to `.env.example`:

```
SENTRY_DSN=
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=
```

---

## Sentry Verification Test

Temporarily add a button or route that throws:

```typescript
throw new Error('Sentry test error');
```

1. Run dev server, hit the throw
2. Open Sentry dashboard → Issues
3. The error should appear within 1 minute
4. Click it — stack trace should show your `.tsx` file (NOT minified `.js`). If minified, source maps aren't uploading; check `SENTRY_AUTH_TOKEN`.
5. Remove the test throw

---

## Drizzle Studio (DB browser for local development)

Drizzle Studio gives you a visual UI for browsing and editing your Neon database rows during development. It is not a monitoring tool but is frequently used alongside Sentry/PostHog to investigate data issues.

### Launch

```bash
npm run db:studio
# or directly:
npx drizzle-kit studio
```

This opens a browser UI at `https://local.drizzle.studio`. It connects to the database specified in your `DATABASE_URL` env var.

### When to use

- After running a migration, to verify the schema looks correct
- When debugging a production bug, to inspect the actual data in your Neon DB
- When seeding test data for development
- To verify that auth, payments, or file upload routes are writing the expected rows

### npm script to add

Add to `package.json` scripts:

```json
{
  "scripts": {
    "db:studio": "drizzle-kit studio"
  }
}
```

### Caution

Drizzle Studio connects directly to whatever `DATABASE_URL` is set in your environment. Be careful not to accidentally run `db:studio` pointed at a production database and edit live data. Always use a dev/test database URL for local development.
