---
name: 4-Build-Monitoring
description: MUST BE USED before any production launch. Wires both Sentry (errors) and PostHog (product analytics) — not one or the other. Walks the user through account setup, env vars, and verification with real test events. Identifies authenticated users in both tools so analytics is attributable. Trigger on `/add-monitoring`, "wire Sentry", "wire PostHog", "add observability".
---

# /add-monitoring

You wire **both** Sentry and PostHog. They don't overlap:
- **Sentry** = production errors + stack traces + performance + source maps
- **PostHog** = product analytics + session replay + feature flags + funnels

If only one is wired, add the other. If both are wired, verify config and exit.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — what's in package.json
- `pattern-finder` — "Find existing analytics/error tracking code: client init, where providers mount, what events are tracked"

### Step 2: Determine state

| Sentry | PostHog | Action |
|---|---|---|
| No | No | Wire both |
| Yes | No | Add PostHog only |
| No | Yes | Add Sentry only |
| Yes | Yes | Verify both; exit |

### Step 3: Account setup (USER does these in browser)

Walk the user through both account setups, verbatim:

> **Sentry** (https://sentry.io):
>
> 1. Sign up for a free account
> 2. Create a new project — when asked "What platform?" pick the framework that matches your project (Next.js, Node.js, React, etc.)
> 3. After creation, you'll see your **DSN** on the setup page — copy it
> 4. Go to **Settings → Account → Auth Tokens** → click **"Create New Token"**
> 5. Scopes needed: `project:read`, `project:releases`, `org:read`
> 6. Copy the token (starts with `sntrys_`) — you won't see it again
> 7. Reply with: DSN, project name, org name, auth token
>
> **PostHog** (https://posthog.com):
>
> 1. Sign up for a free account
> 2. Create a project (or use the default one)
> 3. **Settings → Project → Project API Key** — copy the value (starts with `phc_`)
> 4. Note your host URL — usually `https://us.i.posthog.com` (or `https://eu.i.posthog.com` if EU region)
> 5. Reply with: project API key, host URL

Wait for the user to confirm.

### Step 4: Use Sentry's wizard for installation

For Next.js projects, run:

```bash
npx @sentry/wizard@latest -i nextjs
```

The wizard:
- Installs `@sentry/nextjs`
- Creates `sentry.client.config.ts`, `sentry.server.config.ts`, `sentry.edge.config.ts`
- Wraps `next.config.ts` with `withSentryConfig`
- Creates `instrumentation.ts` if missing
- Asks for DSN and auth token interactively

For other frameworks (Vite, Express), install manually:

```bash
npm install @sentry/node @sentry/react   # Express + React
# or
npm install @sentry/react                # Vite-only
```

Then add `Sentry.init({ dsn: process.env.SENTRY_DSN, tracesSampleRate: 0.1 })` early in the entrypoint.

### Step 5: Wire PostHog

Install:

```bash
npm install posthog-js posthog-node
```

Create `lib/posthog-client.ts` (client) and `lib/posthog-server.ts` (server). Mirror the project's existing lib/* style. The client wraps `<PostHogProvider client={posthog}>` around your root layout. The server uses `posthog-node` for capture-then-shutdown in serverless.

Use `pattern-finder` to find the project's provider-mount pattern (e.g., is there already a providers wrapper in `app/layout.tsx`?). Mount PostHog there.

### Step 6: Identify users on auth events

After successful sign-in (client-side):

```typescript
import * as Sentry from '@sentry/nextjs';
import posthog from 'posthog-js';

Sentry.setUser({ id: user.uid, email: user.email });
posthog.identify(user.uid, { email: user.email, plan: user.plan });
```

On sign-out: `Sentry.setUser(null); posthog.reset();`

### Step 7: Set env vars

Add to `.env.example`:

```
SENTRY_DSN=
NEXT_PUBLIC_SENTRY_DSN=
SENTRY_AUTH_TOKEN=
NEXT_PUBLIC_POSTHOG_KEY=
NEXT_PUBLIC_POSTHOG_HOST=
```

Tell the user to set these in their local `.env.local`. Production env vars get set during `/deploy`.

### Step 8: Verify

> **Test Sentry** — temporarily add a button or route that throws:
>
> ```typescript
> throw new Error('Sentry test error');
> ```
>
> 1. Run dev server, hit the throw
> 2. Open Sentry dashboard → Issues
> 3. The error should appear within 1 minute
> 4. Click it — stack trace should show your `.tsx` file (NOT minified `.js`). If minified, source maps aren't uploading; check `SENTRY_AUTH_TOKEN`.
> 5. Remove the test throw

> **Test PostHog**:
>
> 1. Visit a few pages on dev
> 2. Sign in
> 3. Open PostHog dashboard → Activity → Live events
> 4. Pageview events + identify event should appear within seconds

## Minimum events to track in PostHog

Don't track everything. Track these:

- `signed_up` — new account
- `signed_in` — successful sign-in
- `subscription_started` — first paid event
- `subscription_canceled`
- `feature_used` — for the 3-5 core features (e.g., `chat_message_sent`, `report_generated`)
- `payment_attempted`, `payment_succeeded`, `payment_failed`

Skip: every button click, every form input. Session replay covers granular UX.

## Rules

- **Both, not one.** Sentry alone misses product insight; PostHog alone misses production debugging.
- **Identify users on auth.** Anonymous events are 80% useful; identified events are 100%.
- **Mask password fields in PostHog session replay** (`maskAllInputs: true` is the safe default).
- **Source maps in CI.** Set `SENTRY_AUTH_TOKEN` in CI env so production builds upload source maps automatically. Without this, production stack traces are useless.
- **Don't log PII to Sentry.** Mask emails/names in `beforeSend` if you store sensitive data in error contexts.
