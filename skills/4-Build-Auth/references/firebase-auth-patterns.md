---
title: "Firebase Auth Patterns"
skill: "4-Build-Auth"
---

# Firebase Auth Patterns

## Initial setup (Next.js + Firebase Auth)

```
Files:
  lib/firebase.ts         — client SDK
  lib/firebase-admin.ts   — server SDK (admin)
  middleware.ts           — verify ID token on protected routes
  app/api/auth/session/route.ts — exchange ID token for session cookie (optional)
  app/(auth)/sign-in/page.tsx
  app/(auth)/sign-up/page.tsx
```

## Client SDK

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

## Admin SDK (server)

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

## Token verification middleware

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

## Sign-in page (Next.js client component)

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

## Calling protected APIs from client

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
