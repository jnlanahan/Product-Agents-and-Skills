# Neon Auth Patterns (Better Auth + Next.js App Router)

## What is Neon Auth?

Neon Auth is authentication built on top of **Better Auth** — a TypeScript-first auth library. The key advantage: your user tables live directly inside your Neon Postgres database, so you have one database for everything. No separate Firebase project, no external auth database to sync.

---

## Step-by-step: First-time Setup

### 1. Get your Google OAuth credentials (do this in the browser first)

Before writing any code, the user needs Google OAuth credentials:

1. Go to [console.cloud.google.com](https://console.cloud.google.com)
2. Create a new project (or use an existing one)
3. Navigate to **APIs & Services → Credentials**
4. Click **Create Credentials → OAuth 2.0 Client IDs**
5. Application type: **Web application**
6. Add authorized redirect URI: `http://localhost:3000/api/auth/callback/google` (for dev)
7. Add your production redirect URI too: `https://yourdomain.com/api/auth/callback/google`
8. Copy the **Client ID** and **Client Secret**

### 2. Install dependencies

```bash
npm install better-auth @better-auth/neon
```

### 3. Set environment variables

Add to `.env.local`:

```
BETTER_AUTH_SECRET=       # Long random string — generate: openssl rand -base64 32
BETTER_AUTH_URL=http://localhost:3000
AUTH_GOOGLE_CLIENT_ID=    # From Google Cloud Console
AUTH_GOOGLE_CLIENT_SECRET= # From Google Cloud Console
```

### 4. Wire t3-env validation

```typescript
// src/env.ts
import { createEnv } from '@t3-oss/env-nextjs';
import { z } from 'zod';

export const env = createEnv({
  server: {
    DATABASE_URL: z.string().url(),
    BETTER_AUTH_SECRET: z.string().min(32),
    BETTER_AUTH_URL: z.string().url(),
    AUTH_GOOGLE_CLIENT_ID: z.string().min(1),
    AUTH_GOOGLE_CLIENT_SECRET: z.string().min(1),
  },
  client: {},
  runtimeEnv: {
    DATABASE_URL: process.env.DATABASE_URL,
    BETTER_AUTH_SECRET: process.env.BETTER_AUTH_SECRET,
    BETTER_AUTH_URL: process.env.BETTER_AUTH_URL,
    AUTH_GOOGLE_CLIENT_ID: process.env.AUTH_GOOGLE_CLIENT_ID,
    AUTH_GOOGLE_CLIENT_SECRET: process.env.AUTH_GOOGLE_CLIENT_SECRET,
  },
});
```

### 5. Create the auth config

```typescript
// lib/auth.ts
import { betterAuth } from 'better-auth';
import { drizzleAdapter } from 'better-auth/adapters/drizzle';
import { db } from '@/db'; // your Drizzle db instance
import { env } from '@/env';

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: 'pg',
  }),
  secret: env.BETTER_AUTH_SECRET,
  baseURL: env.BETTER_AUTH_URL,
  socialProviders: {
    google: {
      clientId: env.AUTH_GOOGLE_CLIENT_ID,
      clientSecret: env.AUTH_GOOGLE_CLIENT_SECRET,
    },
  },
  emailAndPassword: {
    enabled: true, // set false if you want Google-only
  },
});

export type Session = typeof auth.$Infer.Session;
```

### 6. Create the API catch-all route

```typescript
// app/api/auth/[...all]/route.ts
import { auth } from '@/lib/auth';
import { toNextJsHandler } from 'better-auth/next-js';

export const { GET, POST } = toNextJsHandler(auth);
```

### 7. Create the auth client (for client components)

```typescript
// lib/auth-client.ts
import { createAuthClient } from 'better-auth/react';

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL ?? 'http://localhost:3000',
});

export const { signIn, signOut, signUp, useSession } = authClient;
```

### 8. Add session middleware for protected routes

```typescript
// middleware.ts (at the repo root, next to package.json)
import { NextRequest, NextResponse } from 'next/server';
import { auth } from '@/lib/auth';

const PROTECTED_PATHS = ['/dashboard', '/settings', '/api/protected'];

export async function middleware(request: NextRequest) {
  const { pathname } = request.nextUrl;
  const isProtected = PROTECTED_PATHS.some(p => pathname.startsWith(p));
  if (!isProtected) return NextResponse.next();

  const session = await auth.api.getSession({ headers: request.headers });
  if (!session) {
    return NextResponse.redirect(new URL('/sign-in', request.url));
  }
  return NextResponse.next();
}

export const config = {
  matcher: ['/dashboard/:path*', '/settings/:path*', '/api/protected/:path*'],
};
```

### 9. Verify session server-side in API routes

```typescript
// app/api/protected/route.ts
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';

export async function GET() {
  const session = await auth.api.getSession({ headers: await headers() });

  if (!session) {
    return new Response('Unauthorized', { status: 401 });
  }

  // session.user.id is the authenticated user's ID
  return Response.json({ userId: session.user.id });
}
```

### 10. Verify session in Server Components

```typescript
// app/dashboard/page.tsx
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';
import { redirect } from 'next/navigation';

export default async function DashboardPage() {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) redirect('/sign-in');

  return <div>Welcome, {session.user.email}</div>;
}
```

---

## Sign-in Page

```typescript
// app/sign-in/page.tsx
'use client';
import { signIn } from '@/lib/auth-client';

export default function SignInPage() {
  return (
    <div>
      <button
        onClick={() => signIn.social({ provider: 'google', callbackURL: '/dashboard' })}
      >
        Sign in with Google
      </button>
    </div>
  );
}
```

For email + password sign-in:

```typescript
'use client';
import { signIn } from '@/lib/auth-client';
import { useState } from 'react';

export default function SignInPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    await signIn.email({ email, password, callbackURL: '/dashboard' });
  }

  return (
    <form onSubmit={handleSubmit}>
      <input type="email" value={email} onChange={e => setEmail(e.target.value)} />
      <input type="password" value={password} onChange={e => setPassword(e.target.value)} />
      <button type="submit">Sign in</button>
    </form>
  );
}
```

---

## Sign-out

```typescript
'use client';
import { signOut } from '@/lib/auth-client';

export function SignOutButton() {
  return (
    <button onClick={() => signOut({ callbackURL: '/' })}>
      Sign out
    </button>
  );
}
```

---

## Reading session in Client Components

```typescript
'use client';
import { useSession } from '@/lib/auth-client';

export function UserGreeting() {
  const { data: session, isPending } = useSession();
  if (isPending) return <p>Loading...</p>;
  if (!session) return <p>Not signed in</p>;
  return <p>Hello, {session.user.email}</p>;
}
```

---

## Database tables (auto-created by Better Auth)

Better Auth automatically creates these tables in your Neon DB on first run:
- `user` — name, email, emailVerified, image, createdAt, updatedAt
- `session` — token, userId, expiresAt, ipAddress, userAgent
- `account` — providerId, providerAccountId, userId (links OAuth accounts)
- `verification` — for email verification tokens

You can extend the `user` table by adding columns in your Drizzle schema and telling Better Auth about them via `additionalFields`.

---

## Environment variables for Vercel

When deploying to Vercel, add these to **Project Settings → Environment Variables**:

```
BETTER_AUTH_SECRET     → your generated secret (same value as local)
BETTER_AUTH_URL        → https://yourdomain.com (your production URL)
AUTH_GOOGLE_CLIENT_ID  → your Google client ID
AUTH_GOOGLE_CLIENT_SECRET → your Google client secret
```

**Important:** Update your Google OAuth redirect URIs in Google Cloud Console to include your production URL: `https://yourdomain.com/api/auth/callback/google`
