---
name: 4-Build-Auth
description: MUST BE USED when the user wants to add or extend authentication. Detects existing auth provider (Firebase Auth, Clerk, NextAuth, Supabase Auth, custom JWT) and adapts to it. If none, scaffolds Firebase Auth per stack preferences. Handles sign-up, sign-in, social login, MFA, organizations, and role-based access. Trigger on `/add-auth`, "add login", "add sign-up", "wire auth".
---

# /add-auth

You add authentication features. Preference is Firebase Auth. If a different auth provider is detected, adapt to it — never migrate.

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

## Firebase Auth Patterns

### Initial setup (Next.js + Firebase Auth)

```
Files:
  lib/firebase.ts         — client SDK
  lib/firebase-admin.ts   — server SDK (admin)
  middleware.ts           — verify ID token on protected routes
  app/api/auth/session/route.ts — exchange ID token for session cookie (optional)
  app/(auth)/sign-in/page.tsx
  app/(auth)/sign-up/page.tsx
```

### Client SDK

```typescript
// lib/firebase.ts
import { initializeApp, getApps, getApp } from 'firebase/app';
import { getAuth } from 'firebase/auth';

const firebaseConfig = {
  apiKey: process.env.NEXT_PUBLIC_FIREBASE_API_KEY!,
  authDomain: process.env.NEXT_PUBLIC_FIREBASE_AUTH_DOMAIN!,
  projectId: process.env.NEXT_PUBLIC_FIREBASE_PROJECT_ID!,
  storageBucket: process.env.NEXT_PUBLIC_FIREBASE_STORAGE_BUCKET!,
  messagingSenderId: process.env.NEXT_PUBLIC_FIREBASE_MESSAGING_SENDER_ID!,
  appId: process.env.NEXT_PUBLIC_FIREBASE_APP_ID!,
};

export const firebaseApp = getApps().length ? getApp() : initializeApp(firebaseConfig);
export const auth = getAuth(firebaseApp);
```

### Admin SDK (server)

```typescript
// lib/firebase-admin.ts
import { initializeApp, getApps, cert, type App } from 'firebase-admin/app';
import { getAuth } from 'firebase-admin/auth';

let app: App;

function getAdminApp() {
  if (getApps().length) return getApps()[0]!;
  const serviceAccount = JSON.parse(process.env.FIREBASE_SERVICE_ACCOUNT!);
  app = initializeApp({ credential: cert(serviceAccount) });
  return app;
}

export const adminAuth = getAuth(getAdminApp());
```

### Token verification middleware

```typescript
// lib/auth.ts
import { adminAuth } from './firebase-admin';

export async function verifyIdToken(req: Request) {
  const authHeader = req.headers.get('authorization') ?? '';
  const token = authHeader.startsWith('Bearer ') ? authHeader.slice(7) : null;
  if (!token) throw new Response('Unauthorized', { status: 401 });
  try {
    return await adminAuth.verifyIdToken(token);
  } catch {
    throw new Response('Unauthorized', { status: 401 });
  }
}
```

### Sign-in page (Next.js client component)

```tsx
'use client';
import { signInWithEmailAndPassword, signInWithPopup, GoogleAuthProvider } from 'firebase/auth';
import { auth } from '@/lib/firebase';

export default function SignInPage() {
  async function handleEmailSignIn(email: string, password: string) {
    await signInWithEmailAndPassword(auth, email, password);
  }
  async function handleGoogleSignIn() {
    await signInWithPopup(auth, new GoogleAuthProvider());
  }
  // ... form UI
}
```

### Calling protected APIs from client

```typescript
const idToken = await auth.currentUser?.getIdToken();
const res = await fetch('/api/me', {
  headers: { Authorization: `Bearer ${idToken}` },
});
```

## Clerk Adaptation (when extending existing Clerk)

```typescript
// Server-side
import { auth, currentUser } from '@clerk/nextjs/server';

const { userId } = await auth();
if (!userId) return new Response('Unauthorized', { status: 401 });
```

```tsx
// Client
import { SignIn, SignUp, UserButton } from '@clerk/nextjs';
```

Adapt all new auth-related code to Clerk's primitives — don't add Firebase alongside.

## NextAuth/Auth.js Adaptation

```typescript
import { auth } from '@/auth';
const session = await auth();
if (!session?.user) return new Response('Unauthorized', { status: 401 });
```

## Custom JWT Adaptation (e.g., kanolens-style)

If the project uses `jose` + Google OAuth manually:
- Read existing `lib/auth.ts` for sign/verify patterns
- Match cookie settings (httpOnly, secure, sameSite, maxAge)
- Mirror the existing JWT claim structure
- Flag if you see security issues but don't change them without explicit user request

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
