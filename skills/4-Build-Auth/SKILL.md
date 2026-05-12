---
name: add-auth
description: MUST BE USED when the user wants to add or extend authentication. Detects existing auth provider (Firebase Auth, Clerk, NextAuth, Supabase Auth, custom JWT) and adapts to it. If none, scaffolds Firebase Auth per stack preferences. Handles sign-up, sign-in, social login, MFA, organizations, and role-based access. Trigger on `/add-auth`, "add login", "add sign-up", "wire auth".
---

# /add-auth

You add authentication features. Preference is Firebase Auth. If a different auth provider is detected, adapt to it — never migrate.

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

### Step 1: Detect

In parallel:
- `stack-detector` — what auth (if any) is detected
- `pattern-finder` — "Find existing auth code: protection middleware, session handling, ID token verification, sign-in page if exists"

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| No auth | Install Firebase Auth (preference) |
| Firebase Auth wired | Extend it |
| Clerk wired | Extend Clerk; do not migrate |
| NextAuth/Auth.js wired | Extend NextAuth |
| Supabase Auth wired | Extend Supabase Auth |
| Custom JWT (e.g., jose + Google OAuth) | Extend custom auth; flag if you spot security issues |

### Step 3: Ask what to add

Use `AskUserQuestion` or prompt:

> What auth feature?
> 1. **Initial setup** (no auth exists yet)
> 2. **Add social login** (Google, GitHub, Apple, etc.)
> 3. **Add MFA** (TOTP, SMS, passkeys)
> 4. **Add organizations / teams** (multi-tenant)
> 5. **Add role-based access** (admin/user/etc.)
> 6. **Add session management** (revoke sessions, list active devices)
> 7. **Add account linking** (let users link Google to existing email account)

### Step 4: Plan

Show concrete changes. Always include:
- Server-side token verification middleware
- Client-side session/state management
- Protected route mechanism
- UI components for sign-in/sign-up if missing

### Step 5: Execute

Write the code, mirroring existing patterns from `pattern-finder`.

### Step 6: Verify end-to-end

- Sign up with a new account → check DB for new user record
- Sign in with same credentials
- Visit a protected page → confirm access
- Sign out → confirm protected page redirects
- For social login: complete the OAuth flow
- For MFA: enroll, sign out, sign in with MFA challenge
- Check Sentry/PostHog for any auth-related errors during the flow

→ See [firebase-auth-patterns.md](references/firebase-auth-patterns.md) for implementation patterns (Firebase client/admin SDK, token verification middleware, sign-in page, Clerk/NextAuth/custom JWT adaptations).

## Common Security Checks

Regardless of provider, verify these in the existing setup OR in your new code:

- [ ] Server-side token verification on every protected route (don't trust client claims)
- [ ] Session cookies set with `httpOnly: true`, `secure: true` in prod, `sameSite: 'lax'` minimum
- [ ] Sign-out actually invalidates the session (clears cookie + revokes refresh token if applicable)
- [ ] Password reset emails use cryptographically random tokens with short expiry
- [ ] Rate limiting on sign-in and sign-up endpoints (prevents brute-force)
- [ ] CSRF protection on state-changing endpoints (or SameSite cookies as primary defense)
- [ ] Firebase: ID tokens have ~1 hour expiry — refresh handled by client SDK automatically
- [ ] Multi-tenant: every DB query scoped to the authenticated user's org/tenant

## Rules

- **Don't migrate auth providers.** This is the most expensive thing to migrate; never do it as a side effect.
- **Verify tokens server-side.** Never trust `req.body.userId` or client-decoded JWT payloads.
- **Match the project's cookie/session conventions** (see `pattern-finder` output).
- **Test the full flow** — sign-up → sign-in → protected route → sign-out → confirm protection. Auth bugs are disasters.
- **For Firebase**: ensure `FIREBASE_SERVICE_ACCOUNT` is the JSON service account key, server-side only, never `NEXT_PUBLIC_*`.

## If Something Goes Wrong

- **OAuth redirect mismatch** — confirm the redirect URI in the provider console exactly matches the one in your env var, including protocol and trailing slash.
- **Social login returns token but user is not created** — check that the provider callback handler writes to the user table; log the raw token to confirm identity data is present.
- **Session not persisting after page reload** — confirm the session cookie is `httpOnly`, `secure` (in production), and `sameSite=lax`; check the cookie domain matches the app domain.
- **MFA flow breaks existing sessions** — existing sessions before MFA was enabled may need to be invalidated; add a `mfa_enrolled_at` field and check it on session resume.