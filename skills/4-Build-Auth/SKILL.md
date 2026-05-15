---
name: add-auth
description: MUST BE USED when the user wants to add or extend authentication. Detects existing auth provider (Neon Auth / Better Auth, Firebase Auth, Clerk, NextAuth, Supabase Auth, custom JWT) and adapts to it. If none, scaffolds Neon Auth via Better Auth per stack preferences. Handles sign-up, sign-in, Google Sign-In, MFA, organizations, and role-based access. Trigger on `/add-auth`, "add login", "add sign-up", "wire auth".
---

# /add-auth

You add authentication features. The default is **Neon Auth (via Better Auth)**. If a different auth provider is already wired, adapt to it — never migrate.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- Never migrate between auth providers in this skill — if one is already wired, extend it or stop and escalate.
- Do not modify session handling or token logic without reading the existing implementation first; breaking auth locks users out immediately.
- Test login, logout, and session expiry end-to-end before committing — a partial auth wiring is worse than none.

## Procedure

### Step 0: Check prerequisites (beginners — read this first)

If the user is setting up auth for the first time, ask:

> Before we write any code, let's confirm two things:
>
> 1. **Do you have a Neon project?** (Neon Auth stores user tables inside your Neon Postgres DB.) If not, run `/setup-database` first.
> 2. **Do you have Google OAuth credentials?** You'll need a `CLIENT_ID` and `CLIENT_SECRET` from the Google Cloud Console. If not, I'll walk you through creating them.
>
> Reply "yes" to both, or let me know which one you need help with first.

Wait for the user before writing any code.

### Step 1: Detect

In parallel:
- `stack-detector` — what auth (if any) is detected
- `pattern-finder` — "Find existing auth code: protection middleware, session handling, token verification, sign-in page"

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| No auth | Install Neon Auth via Better Auth (default) |
| Better Auth / Neon Auth wired | Extend it |
| Firebase Auth wired | Extend Firebase Auth; do not migrate |
| Clerk wired | Extend Clerk; do not migrate |
| NextAuth / Auth.js wired | Extend NextAuth; do not migrate |
| Supabase Auth wired | Extend Supabase Auth; do not migrate |
| Custom JWT (e.g., jose + Google OAuth) | Extend custom auth; flag security issues if spotted |

### Step 3: Ask what to add

> What auth feature do you need?
> 1. **Initial setup** (no auth exists yet)
> 2. **Add Google Sign-In** (OAuth with Google)
> 3. **Add email + password sign-up/sign-in**
> 4. **Add MFA** (two-factor authentication)
> 5. **Add organizations / teams** (multi-tenant)
> 6. **Add role-based access** (admin/user/etc.)
> 7. **Add session management** (revoke sessions, list active devices)
> 8. **Add account linking** (let users link Google to an existing email account)

### Step 4: Plan

Show concrete changes. Always include:
- Server-side session verification middleware
- Client-side session state management
- Protected route mechanism
- UI components for sign-in/sign-up if missing

### Step 5: Execute

Write the code, mirroring existing patterns from `pattern-finder`.

→ See [neon-auth-patterns.md](references/neon-auth-patterns.md) for implementation patterns (Better Auth config, Google Sign-In, session middleware, protected routes, t3-env integration).

→ See [firebase-auth-patterns.md](references/firebase-auth-patterns.md) for Firebase Auth, Clerk, and NextAuth adaptation patterns (for existing projects that use those providers).

### Step 6: Verify end-to-end

- Sign up with a new account → check DB (`users` table in your Neon DB) for the new user record
- Sign in with same credentials
- Visit a protected page → confirm access
- Sign out → confirm protected page redirects to login
- For Google Sign-In: complete the full OAuth flow
- For MFA: enroll, sign out, sign in with MFA challenge
- Check Sentry for any auth-related errors during the flow

## Common Security Checks

Regardless of provider, verify these in the existing setup OR in your new code:

- [ ] Server-side session verification on every protected route (never trust client-side claims)
- [ ] Session cookies set with `httpOnly: true`, `secure: true` in prod, `sameSite: 'lax'` minimum
- [ ] Sign-out actually invalidates the session (clears cookie + revokes session server-side)
- [ ] Password reset emails use cryptographically random tokens with short expiry
- [ ] Rate limiting on sign-in and sign-up endpoints (prevents brute-force)
- [ ] CSRF protection on state-changing endpoints (or `SameSite` cookies as primary defense)
- [ ] Multi-tenant: every DB query scoped to the authenticated user's org/tenant
- [ ] `BETTER_AUTH_SECRET` is a long random string, never committed to git, only in `.env.local`

## Rules

- **Don't migrate auth providers.** This is the most expensive thing to migrate; never do it as a side effect.
- **Verify sessions server-side.** Never trust client-decoded session data alone.
- **Match the project's cookie/session conventions** (see `pattern-finder` output).
- **Test the full flow** — sign-up → sign-in → protected route → sign-out → confirm protection. Auth bugs are disasters.
- **Keep secrets server-side only.** `BETTER_AUTH_SECRET`, Google client secret — never `NEXT_PUBLIC_*`.

## If Something Goes Wrong

- **OAuth redirect mismatch** — confirm the redirect URI in the Google Cloud Console exactly matches `BETTER_AUTH_URL + /api/auth/callback/google`, including protocol and trailing slash.
- **Google Sign-In returns token but user is not created** — check that the callback handler writes to the `users` table; log the raw session to confirm identity data is present.
- **Session not persisting after page reload** — confirm the session cookie is `httpOnly`, `secure` (in production), and `sameSite=lax`; check the cookie domain matches the app domain.
- **MFA flow breaks existing sessions** — existing sessions before MFA was enabled may need to be invalidated; add a `mfa_enrolled_at` field and check it on session resume.
- **`BETTER_AUTH_SECRET` not set** — Better Auth will throw a startup error; generate with `openssl rand -base64 32` and add to `.env.local`.
