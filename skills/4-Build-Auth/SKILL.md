---
name: 4-Build-Auth
description: MUST BE USED when the user wants to add or extend authentication. Detects existing auth provider (Neon Auth / Better Auth, Firebase Auth, Clerk, NextAuth, Supabase Auth, custom JWT) and adapts to it. If none, scaffolds Neon Auth via Better Auth per stack preferences. Handles sign-up, sign-in, Google Sign-In, MFA, organizations, and role-based access.
when_to_use: "User says 'add login', 'add sign-up', 'wire auth', 'add authentication', 'I need users to sign in', 'add Google Sign-In', 'set up auth'."
---

# /4-Build-Auth

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
> 1. **Do you have a Neon project?** (Neon Auth stores user tables inside your Neon Postgres DB.) If not, run `/4-Build-Database` first.
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

→ See [neon-auth-patterns.md](references/neon-auth-patterns.md) for implementation patterns (hand-written Drizzle auth schema, Better Auth config, Google Sign-In, email-verification sign-up state, password reset, session middleware, protected routes, optional t3-env).

→ See [firebase-auth-patterns.md](references/firebase-auth-patterns.md) for Firebase Auth, Clerk, and NextAuth adaptation patterns (for existing projects that use those providers).

**Windows / tsx env setup (do this during setup, not after debugging):**
`--env-file` is unreliable on Windows with `tsx`. Instead:
1. `npm install dotenv`
2. Add `import "dotenv/config"` as the very first line of the server entry point.

**After adding `GOOGLE_CLIENT_ID` and `GOOGLE_CLIENT_SECRET` to `.env`:** `tsx watch` does not auto-restart on `.env` changes. Tell the user to stop and restart the server.

**Windows port-orphan pitfall (Next.js):** a backgrounded `next dev` on Windows frequently orphans a node process that keeps holding the port after you think you've stopped it. The symptom is Next.js silently starting on `3001`/`3002` (which then breaks `BETTER_AUTH_URL` and the Google redirect URI), or `EADDRINUSE`. Don't chase it — kill the port and restart cleanly:
```powershell
# find and kill whatever is on port 3000, then restart
npx kill-port 3000   # or: Get-NetTCPConnection -LocalPort 3000 | ForEach-Object { Stop-Process -Id $_.OwningProcess -Force }
npm run dev
```
Prefer running `next dev` in the foreground while testing auth so the port is released the moment you stop it.

**Google Auth Platform setup (console UI as of 2025):**
The old "OAuth consent screen" is now **Google Auth Platform** with a sidebar:
- **Branding** — app name, logo, support email
- **Audience** — internal vs. external; add test users here
- **Clients** — create the OAuth client and get your credentials

When creating the OAuth client, two URI fields trip people up:
- **Authorized JavaScript origins** — domain only, no path: `http://localhost:3001`
- **Authorized redirect URIs** — full callback path: `http://localhost:3001/api/auth/callback/google`

The JavaScript origins field rejects paths — it will silently error or fail if you include one. State this explicitly before the user fills it in.

### Step 6: Verify end-to-end

**Fast wiring check with curl (do this before any manual clicking).** These hit the Better Auth endpoints directly and confirm the routes, DB, and providers are wired without a browser round-trip. Replace `3000` with the actual dev-server port.

```bash
# 1. Email sign-up — expect 200 with a user object (or a "check email" response if verification is on)
curl -i -X POST http://localhost:3000/api/auth/sign-up/email \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password1234","name":"Test"}'

# 2. Email sign-in — expect 200 + a set-cookie session header
curl -i -X POST http://localhost:3000/api/auth/sign-in/email \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","password":"password1234"}'

# 3. Google social sign-in — expect 200 with a JSON { url: "https://accounts.google.com/..." } redirect target
curl -i -X POST http://localhost:3000/api/auth/sign-in/social \
  -H "Content-Type: application/json" \
  -d '{"provider":"google","callbackURL":"/dashboard"}'

# 4. Password reset request — expect 200 (and, if email is wired, a Resend delivery)
curl -i -X POST http://localhost:3000/api/auth/request-password-reset \
  -H "Content-Type: application/json" \
  -d '{"email":"test@example.com","redirectTo":"/reset-password"}'
```

A `404` on any of these means the catch-all route isn't wired; a `500` usually means the DB tables weren't pushed (`npm run db:push`) or `BETTER_AUTH_SECRET` is missing. Once curl is green, do the manual flow below.

- **Health check first:** Before testing the OAuth flow, hit `/api/health` (or equivalent) and confirm `google_oauth: true` and `database: true` are both present. This catches a misconfigured env before a confusing OAuth round-trip.
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
