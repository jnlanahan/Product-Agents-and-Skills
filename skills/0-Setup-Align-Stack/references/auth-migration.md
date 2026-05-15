# Auth Migration Reference — Any Auth Provider → Neon Auth via Better Auth

Referenced by: [../SKILL.md](../SKILL.md) (Auth wave)

Covers migration from:
- **Clerk** → Better Auth
- **Supabase Auth** → Better Auth
- **NextAuth (Auth.js v4/v5)** → Better Auth
- **Firebase Auth** → Better Auth

---

## Before You Start: The User Guardrail

**Answer this question first:** Are there real users with live accounts in the current auth system?

- **No** (dev/staging/early launch; 0–10 users willing to re-register): proceed with the clean swap in this doc.
- **Yes** (production users, real data): do NOT simply swap providers. Choose a strategy from "Migrating Live User Accounts" at the bottom of this doc before writing any code.

---

## Better Auth Baseline Setup (all scenarios)

### Install

```bash
npm install better-auth
```

### auth.ts (project root)

```ts
import { betterAuth } from 'better-auth';
import { Pool } from '@neondatabase/serverless';

export const auth = betterAuth({
  database: new Pool({ connectionString: process.env.DATABASE_URL }),
  socialProviders: {
    google: {
      clientId: process.env.AUTH_GOOGLE_CLIENT_ID!,
      clientSecret: process.env.AUTH_GOOGLE_CLIENT_SECRET!,
    },
  },
});
```

Better Auth auto-creates its own tables (`user`, `session`, `account`, `verification`) in the Neon DB on first request — no manual schema needed.

### API route

```ts
// app/api/auth/[...all]/route.ts
import { auth } from '@/auth';
import { toNextJsHandler } from 'better-auth/next-js';

export const { GET, POST } = toNextJsHandler(auth.handler);
```

### Server-side session helper

```ts
// lib/auth-server.ts
import { auth } from '@/auth';
import { headers } from 'next/headers';

export async function getSession() {
  return auth.api.getSession({ headers: await headers() });
}
```

### Client-side auth client

```ts
// lib/auth-client.ts
import { createAuthClient } from 'better-auth/react';

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL,
});
```

### Sign-in trigger (client component)

```ts
await authClient.signIn.social({ provider: 'google', callbackURL: '/dashboard' });
```

### Sign-out

```ts
await authClient.signOut({ fetchOptions: { onSuccess: () => router.push('/') } });
```

### Required env vars

```
BETTER_AUTH_SECRET=          # openssl rand -base64 32
BETTER_AUTH_URL=             # http://localhost:3000 (no trailing slash)
AUTH_GOOGLE_CLIENT_ID=
AUTH_GOOGLE_CLIENT_SECRET=
NEXT_PUBLIC_APP_URL=         # same as BETTER_AUTH_URL; used by auth-client.ts
```

---

## From Clerk

### What to remove

```bash
npm uninstall @clerk/nextjs @clerk/clerk-sdk-node @clerk/backend
```

Delete or clean:
- `<ClerkProvider>` wrapper from `app/layout.tsx`
- `authMiddleware` or `clerkMiddleware` export from `middleware.ts`
- `NEXT_PUBLIC_CLERK_PUBLISHABLE_KEY`, `CLERK_SECRET_KEY` from `.env`

### Replacement mapping

| Clerk | Better Auth |
|---|---|
| `<ClerkProvider>` | No wrapper needed (API-based sessions) |
| `authMiddleware({ publicRoutes })` | Custom middleware — see below |
| `clerkMiddleware()` | Custom middleware — see below |
| `currentUser()` (server component) | `getSession()` from `lib/auth-server.ts` |
| `auth()` (server action / route) | `getSession()` from `lib/auth-server.ts` |
| `useUser()` (client) | `authClient.useSession()` |
| `useAuth()` (client) | `authClient.useSession()` |
| `<UserButton />` | Build a custom avatar/dropdown using `authClient.useSession()` |
| `<SignInButton />` | `authClient.signIn.social({ provider: 'google' })` |
| `<SignUpButton />` | Same — Google OAuth handles both sign-in and sign-up |

### Middleware replacement

```ts
// middleware.ts
import { NextRequest, NextResponse } from 'next/server';
import { auth } from '@/auth';

const PUBLIC_PATHS = ['/', '/sign-in', '/api/auth'];

export async function middleware(req: NextRequest) {
  const isPublic = PUBLIC_PATHS.some(p => req.nextUrl.pathname.startsWith(p));
  if (isPublic) return NextResponse.next();

  const session = await auth.api.getSession({ headers: req.headers });
  if (!session) return NextResponse.redirect(new URL('/sign-in', req.url));
  return NextResponse.next();
}

export const config = {
  matcher: ['/((?!_next/static|_next/image|favicon.ico).*)'],
};
```

### Data note

Clerk does not export hashed passwords — email/password accounts cannot be migrated automatically. If users have **only** Google (or other OAuth) accounts, see "Strategy C — Manual user seeding" below to pre-seed by email.

---

## From Supabase Auth

### What to remove

```bash
npm uninstall @supabase/auth-helpers-nextjs @supabase/ssr @supabase/supabase-js
```

Remove env vars: `NEXT_PUBLIC_SUPABASE_URL`, `NEXT_PUBLIC_SUPABASE_ANON_KEY`, `SUPABASE_SERVICE_ROLE_KEY`
(Keep `DATABASE_URL` if you're migrating the Supabase DB to Neon in the same run.)

### Replacement mapping

| Supabase Auth | Better Auth |
|---|---|
| `createServerComponentClient()` → `supabase.auth.getSession()` | `getSession()` from `lib/auth-server.ts` |
| `createRouteHandlerClient()` → `supabase.auth.getUser()` | `getSession()` — `session.user` has the user |
| `supabase.auth.signInWithOAuth({ provider: 'google' })` | `authClient.signIn.social({ provider: 'google' })` |
| `supabase.auth.signOut()` | `authClient.signOut()` |
| `supabase.auth.onAuthStateChange()` | `authClient.useSession()` — reactive |
| `createMiddlewareClient()` + `supabase.auth.getSession()` | Better Auth middleware — see Clerk section above |

### Data note

Supabase's `auth.users` table stores user records. Passwords are bcrypt-hashed and cannot be ported. Social OAuth users can be migrated by email match — see "Strategy C — Manual user seeding."

The Supabase `auth` schema (tables like `auth.users`, `auth.sessions`) is internal and should NOT be copied to Neon. Better Auth creates its own clean schema.

---

## From NextAuth (Auth.js v4 / v5)

### What to remove

```bash
npm uninstall next-auth @auth/core @auth/prisma-adapter @auth/drizzle-adapter
```

Delete:
- `app/api/auth/[...nextauth]/route.ts` (replaced by Better Auth's route)
- `authOptions` config object
- `<SessionProvider>` wrapper in `layout.tsx`

### Replacement mapping

| NextAuth | Better Auth |
|---|---|
| `getServerSession(authOptions)` | `getSession()` from `lib/auth-server.ts` |
| `useSession()` from `next-auth/react` | `authClient.useSession()` from `lib/auth-client.ts` |
| `<SessionProvider>` | No wrapper needed |
| `authOptions.providers` array | `socialProviders` in `auth.ts` |
| `authOptions.adapter` | `database` in `auth.ts` (Better Auth manages its own schema) |
| `authOptions.callbacks.session` | Better Auth `plugins` / `hooks` API |
| `authOptions.callbacks.jwt` | Better Auth uses database sessions, not JWTs by default |

### Schema note

NextAuth stores data in `users`, `accounts`, `sessions`, `verification_tokens` tables. Better Auth uses different table names. If you have existing NextAuth user data:
- `users.email` → maps to Better Auth `user.email`
- `users.name` → maps to Better Auth `user.name`
- `users.image` → maps to Better Auth `user.image`
- `accounts` → Better Auth manages OAuth account linking internally

For a clean swap (no real users), delete the old NextAuth tables after Better Auth creates its own.

---

## From Firebase Auth

### What to remove

```bash
npm uninstall firebase firebase-admin
```

Remove env vars: `NEXT_PUBLIC_FIREBASE_API_KEY`, `NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN`, `FIREBASE_PROJECT_ID`, `FIREBASE_PRIVATE_KEY`, etc.

### Replacement mapping

| Firebase Auth | Better Auth |
|---|---|
| `onAuthStateChanged(auth, user => ...)` | `authClient.useSession()` — reactive |
| `signInWithPopup(auth, new GoogleAuthProvider())` | `authClient.signIn.social({ provider: 'google' })` |
| `signOut(auth)` | `authClient.signOut()` |
| `auth.currentUser` | `authClient.useSession().data?.user` |
| `getIdToken(user)` | Not needed — Better Auth sessions are DB-backed |
| `getAuth(adminApp).verifyIdToken(token)` | `auth.api.getSession({ headers })` — server-side DB lookup |
| Firebase Admin SDK user lookup | `auth.api.getSession()` or direct Drizzle query on the `user` table |

### Data note

Firebase Auth tokens are Firebase-proprietary JWTs — they cannot be migrated. All existing users must sign in again after the swap. Firebase does allow user export (`firebase auth:export users.json`) which includes email, display name, and photo URL but not passwords. Use this export for "Strategy C — Manual user seeding."

---

## Migrating Live User Accounts

If real production users exist, choose one of these strategies. None of them are automated — each requires manual steps.

### Strategy A: Hard Cut with Re-Registration

**Best for:** < 50 users; users agree to re-auth; early-stage product.

1. Announce to users: "We're upgrading our sign-in system. On [date], please sign in with Google. Your account data is preserved."
2. On migration date: swap to Better Auth, redirect all `/sign-in` routes to Google OAuth
3. Better Auth creates new user records on first OAuth sign-in
4. After sign-in, match `session.user.email` to the old user record and migrate any user-owned data rows to the new `userId`
5. Write a one-time migration script that runs on first sign-in: `UPDATE posts SET user_id = $newId WHERE user_id = $oldId`

### Strategy B: Parallel Run (Zero Downtime)

**Best for:** > 50 users; cannot afford forced re-registration.

1. Keep old auth provider running. Do NOT remove it.
2. Wire Better Auth alongside it. Both auth systems are live simultaneously.
3. New sign-ins go through Better Auth. Existing sessions continue through old auth.
4. On each old-auth session, prompt: "Upgrade your account — sign in with Google to link your account." This creates a Better Auth record linked by email.
5. After 30 days (or when % of linked accounts is high enough), decommission old auth.
6. Users who never re-linked lose their session and must sign in fresh through Better Auth.

### Strategy C: Manual User Seeding (for OAuth-only accounts)

**Best for:** All users signed up via Google (no passwords); you have their email list.

1. Export user list from old auth provider (email + name + profile image URL)
2. Pre-create Better Auth user records via the admin API:
   ```ts
   // Run as a one-time script with admin credentials
   await auth.api.createUser({
     body: {
       email: 'user@example.com',
       name: 'User Name',
       emailVerified: true,
     },
   });
   ```
3. Send all users a "sign in with Google" email — they match by email on first OAuth sign-in
4. After 30 days, audit which emails signed in and which did not; decommission old auth

### What NOT to Do

- Do NOT copy hashed passwords between systems — hash algorithms and salting differ; re-hashed passwords will not match on login
- Do NOT delete old user accounts before confirming new sessions work end-to-end
- Do NOT decommission old auth before a parallel window of at least 14 days
- Do NOT expose the migration script as a public API endpoint — run it as a server-side admin script or from a protected route
