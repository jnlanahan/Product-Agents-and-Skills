---
name: 4-Build-Monitoring
description: MUST BE USED before any production launch. Wires both Sentry (errors) and PostHog (product analytics) — not one or the other. Walks the user through account setup, env vars, and verification with real test events. Identifies authenticated users in both tools so analytics is attributable.
when_to_use: "User says 'wire Sentry', 'wire PostHog', 'add observability', 'add error tracking', 'add analytics', 'set up monitoring'."
---

# /4-Build-Monitoring

You wire **both** Sentry and PostHog. They don't overlap:
- **Sentry** = production errors + stack traces + performance + source maps
- **PostHog** = product analytics + session replay + feature flags + funnels

If only one is wired, add the other. If both are wired, verify config and exit.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Both Sentry AND PostHog must be wired — do not deliver half the monitoring stack and call it done.
- Verify DSN/API keys are set in the environment before finishing; SDKs fail silently if keys are missing.
- Identify the authenticated user in both tools from the start — anonymous events are hard to act on post-launch.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — what's in package.json
- `pattern-finder` — "Find existing analytics/error tracking code: client init, where providers mount, what events are tracked. Also find: error boundaries, onError handlers, stream/async failure paths, auth sign-in/sign-out handlers, and the 3–5 core feature actions the app performs."

After both agents return, generate a short "Here's what we'll monitor" summary tailored to this app before doing anything else. Example shape:

> **What Sentry will catch in your app:** server crashes in your API routes, React render errors in [ErrorBoundary], and failures in [specific async path like stream processing or file upload].
>
> **What PostHog will track:** who [does the core action], how far users get in [specific funnel], and where they drop off. We'll fire events like `[feature]_started`, `[feature]_completed`, `signed_in`.

Show this to the user before continuing. This sets expectations so nothing surprises them mid-setup.

### Step 2: Determine state

| Sentry | PostHog | Action |
|---|---|---|
| No | No | Wire both |
| Yes | No | Add PostHog only |
| No | Yes | Add Sentry only |
| Yes | Yes | Verify both; exit |

### Step 3: Account setup (USER does these in browser)

Before asking for credentials, give a one-paragraph plain-English description of what each tool will do in **this specific app** — not generic marketing copy. Use the pattern-finder results from Step 1. For example:

> "Sentry will automatically catch crashes in your API routes, React render errors in your ErrorBoundary, and any unhandled promise rejections. When something breaks in production you'll get an email with the exact file and line number.
>
> PostHog will record every time a user starts an analysis, how far they get, and whether they complete it. You can see session replays of confusing moments and build a funnel to see where people drop off."

Then walk through account setup:

> **Sentry** (https://sentry.io):
>
> 1. Sign up for a free account
> 2. Create a new project — when asked "What platform?" pick the framework that matches your project (Next.js, Node.js, React, etc.)
> 3. After creation, you'll see your **DSN** on the setup page — copy it
> 4. For the auth token: go to your **profile avatar (bottom-left) → User Settings → Auth Tokens** OR **Settings → [Your Org] → Organization Auth Tokens** — you want the **Organization Auth Token**, NOT the Security Token shown in project settings
> 5. Click **"Create New Token"**; scopes needed: `project:read`, `project:releases`, `org:read`
> 6. Copy the token (starts with `sntrys_`) — you won't see it again
>
> **Paste both the DSN and the auth token directly in this chat — I'll add them to your .env file.**
>
> Also tell me your Sentry project name and org name (shown in the URL: `sentry.io/organizations/<org-name>/`).

> **PostHog** (https://posthog.com):
>
> **Free tier note:** PostHog's free plan allows 1 project. If you have multiple apps, use a single project and we'll tag every event with `app: 'yourappname'` so you can filter per app in dashboards — no upgrade needed.
>
> 1. Sign up for a free account
> 2. Create a project (or use the default one)
> 3. Go to **Settings → Project → Project API Key** — copy the value (starts with `phc_`)
> 4. Note your host URL — usually `https://us.i.posthog.com` (or `https://eu.i.posthog.com` if EU region)
>
> **Paste the API key and host URL directly in this chat — I'll add them to your .env file.**

Wait for the user to provide all values before continuing.

→ See [sentry-posthog-patterns.md](references/sentry-posthog-patterns.md) for installation commands, user identification code, env var list, and Sentry verification test snippet.

### Step 4: Install Sentry

For Next.js, run `npx @sentry/wizard@latest -i nextjs` — it handles config files, `next.config.ts` wrapping, and `instrumentation.ts`. For Vite/Express, install `@sentry/react` or `@sentry/node` manually and call `Sentry.init()` early in the entrypoint.

### Step 5: Wire PostHog

Install `posthog-js posthog-node`. Create `lib/posthog-client.ts` and `lib/posthog-server.ts` mirroring the project's lib/* style. Use `pattern-finder` to find the existing providers wrapper in `app/layout.tsx` and mount PostHog there.

### Step 6: Identify users on auth events

Call `Sentry.setUser()` and `posthog.identify()` after successful sign-in; call `Sentry.setUser(null)` and `posthog.reset()` on sign-out. See reference file for the exact snippet.

### Step 7: Set env vars

Add `SENTRY_DSN`, `NEXT_PUBLIC_SENTRY_DSN`, `SENTRY_AUTH_TOKEN`, `NEXT_PUBLIC_POSTHOG_KEY`, and `NEXT_PUBLIC_POSTHOG_HOST` to `.env.example`. Write the actual values to `.env.local`.

> **Important:** After writing the env vars, tell the user: "Restart the dev server now — Vite/Next.js won't pick up new env vars without a full restart. Stop the server (Ctrl+C) and run your start command again before testing."

Production vars get set during `/6-Deploy`.

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

- **Sentry events not appearing / stuck on "waiting"** — the most common cause is the dev server wasn't restarted after adding env vars. Stop it (Ctrl+C) and rerun. Then confirm DSN is present in `.env.local`. Sentry also has a test button: Settings → Projects → your project → "Send a test event".
- **PostHog not tracking events** — check the API key and host in the PostHog config; confirm `posthog.identify()` is called after login and not before the client initializes.
- **User identity not showing in Sentry/PostHog** — confirm `Sentry.setUser()` and `posthog.identify()` are called after authentication resolves, not on app mount.
- **Source maps not uploading to Sentry** — verify `SENTRY_AUTH_TOKEN` and `SENTRY_ORG`/`SENTRY_PROJECT` are set in CI; source maps must be uploaded at build time, not runtime.