---
name: add-monitoring
description: MUST BE USED before any production launch. Wires both Sentry (errors) and PostHog (product analytics) — not one or the other. Walks the user through account setup, env vars, and verification with real test events. Identifies authenticated users in both tools so analytics is attributable. Trigger on `/add-monitoring`, "wire Sentry", "wire PostHog", "add observability".
---

# /add-monitoring

You wire **both** Sentry and PostHog. They don't overlap:
- **Sentry** = production errors + stack traces + performance + source maps
- **PostHog** = product analytics + session replay + feature flags + funnels

If only one is wired, add the other. If both are wired, verify config and exit.

## Important

- Both Sentry AND PostHog must be wired — do not deliver half the monitoring stack and call it done.
- Verify DSN/API keys are set in the environment before finishing; SDKs fail silently if keys are missing.
- Identify the authenticated user in both tools from the start — anonymous events are hard to act on post-launch.

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

→ See [sentry-posthog-patterns.md](references/sentry-posthog-patterns.md) for installation commands, user identification code, env var list, and Sentry verification test snippet.

### Step 4: Install Sentry

For Next.js, run `npx @sentry/wizard@latest -i nextjs` — it handles config files, `next.config.ts` wrapping, and `instrumentation.ts`. For Vite/Express, install `@sentry/react` or `@sentry/node` manually and call `Sentry.init()` early in the entrypoint.

### Step 5: Wire PostHog

Install `posthog-js posthog-node`. Create `lib/posthog-client.ts` and `lib/posthog-server.ts` mirroring the project's lib/* style. Use `pattern-finder` to find the existing providers wrapper in `app/layout.tsx` and mount PostHog there.

### Step 6: Identify users on auth events

Call `Sentry.setUser()` and `posthog.identify()` after successful sign-in; call `Sentry.setUser(null)` and `posthog.reset()` on sign-out. See reference file for the exact snippet.

### Step 7: Set env vars

Add `SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `NEXT_PUBLIC_POSTHOG_KEY`, and `NEXT_PUBLIC_POSTHOG_HOST` to `.env.example`. Tell the user to set these in `.env.local`. Production vars get set during `/deploy`.

### Step 8: Verify

> **Test Sentry** — temporarily throw `new Error('Sentry test error')` in a route, hit it, confirm the issue appears in the Sentry dashboard within 1 minute with a readable `.tsx` stack trace (not minified). Remove the throw after confirming.

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

## If Something Goes Wrong

- **Sentry events not appearing** — confirm the DSN is set in `.env` and the server was restarted; use Sentry's test event button (Settings > Projects > your project > "Send a test event").
- **PostHog not tracking events** — check the API key and host in the PostHog config; confirm `posthog.identify()` is called after login and not before the client initializes.
- **User identity not showing in Sentry/PostHog** — confirm `Sentry.setUser()` and `posthog.identify()` are called after authentication resolves, not on app mount.
- **Source maps not uploading to Sentry** — verify `SENTRY_AUTH_TOKEN` and `SENTRY_ORG`/`SENTRY_PROJECT` are set in CI; source maps must be uploaded at build time, not runtime.