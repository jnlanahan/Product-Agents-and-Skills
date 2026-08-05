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
npm install better-auth
```

> There is **no `@better-auth/neon` package** — Better Auth talks to Neon through the standard Drizzle (or Postgres) adapter. Installing `@better-auth/neon` will fail. You only need `better-auth` plus your existing Drizzle setup.

**Port note:** the examples below use `http://localhost:3000`. If your dev server runs on a different port (Next.js auto-bumps to `3001`, `3002`, … when `3000` is taken), replace **every** `3000` here — in `BETTER_AUTH_URL`, the auth-client `baseURL`, and the Google redirect URI — with your actual port. A port mismatch is the most common cause of a silent OAuth failure.

### 3. Set environment variables

Add to `.env.local`:

```
BETTER_AUTH_SECRET=       # Long random string — generate: openssl rand -base64 32
BETTER_AUTH_URL=http://localhost:3000
AUTH_GOOGLE_CLIENT_ID=    # From Google Cloud Console
AUTH_GOOGLE_CLIENT_SECRET= # From Google Cloud Console
```

### 4. (Optional) Wire t3-env validation

**t3-env is optional, not required.** If the project doesn't already use `@t3-oss/env-nextjs`, skip this step and read directly from `process.env` in `lib/auth.ts` (with a `!` or a manual check). Only add t3-env if the project already has it wired or the user asks for env validation. Do not install it just for auth.

If you do use it:

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

### 5. Hand-write the Drizzle auth schema (do NOT rely on the CLI generator)

The Better Auth CLI generator (`npx @better-auth/cli generate`) **cannot resolve `@/` path aliases**, so it fails on most Next.js projects that import `db` via `@/db`. Hand-write the schema instead. Paste this as-is — it matches Better Auth's expected table/column names for the Postgres + Drizzle adapter:

```typescript
// db/auth-schema.ts
import { pgTable, text, timestamp, boolean } from 'drizzle-orm/pg-core';

export const user = pgTable('user', {
  id: text('id').primaryKey(),
  name: text('name').notNull(),
  email: text('email').notNull().unique(),
  emailVerified: boolean('email_verified').notNull().default(false),
  image: text('image'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});

export const session = pgTable('session', {
  id: text('id').primaryKey(),
  expiresAt: timestamp('expires_at').notNull(),
  token: text('token').notNull().unique(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
  ipAddress: text('ip_address'),
  userAgent: text('user_agent'),
  userId: text('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
});

export const account = pgTable('account', {
  id: text('id').primaryKey(),
  accountId: text('account_id').notNull(),
  providerId: text('provider_id').notNull(),
  userId: text('user_id').notNull().references(() => user.id, { onDelete: 'cascade' }),
  accessToken: text('access_token'),
  refreshToken: text('refresh_token'),
  idToken: text('id_token'),
  accessTokenExpiresAt: timestamp('access_token_expires_at'),
  refreshTokenExpiresAt: timestamp('refresh_token_expires_at'),
  scope: text('scope'),
  password: text('password'),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});

export const verification = pgTable('verification', {
  id: text('id').primaryKey(),
  identifier: text('identifier').notNull(),
  value: text('value').notNull(),
  expiresAt: timestamp('expires_at').notNull(),
  createdAt: timestamp('created_at').notNull().defaultNow(),
  updatedAt: timestamp('updated_at').notNull().defaultNow(),
});
```

Then push it to the DB: `npm run db:push` (or your project's Drizzle migration command). **Tell the user to run this before testing — the app will error on first sign-up if the tables don't exist.**

### 6. Create the auth config

Use t3-env (`env`) **only if you wired it in step 4**; otherwise read straight from `process.env` as shown in the commented alternative.

```typescript
// lib/auth.ts
import { betterAuth } from 'better-auth';
import { drizzleAdapter } from 'better-auth/adapters/drizzle';
import { db } from '@/db'; // your Drizzle db instance

export const auth = betterAuth({
  database: drizzleAdapter(db, {
    provider: 'pg',
  }),
  secret: process.env.BETTER_AUTH_SECRET!,
  baseURL: process.env.BETTER_AUTH_URL!,
  socialProviders: {
    google: {
      clientId: process.env.AUTH_GOOGLE_CLIENT_ID!,
      clientSecret: process.env.AUTH_GOOGLE_CLIENT_SECRET!,
    },
  },
  emailAndPassword: {
    enabled: true, // set false if you want Google-only
    // requireEmailVerification: true,  // see "Email verification" below before enabling
  },
});

export type Session = typeof auth.$Infer.Session;
```

### 7. Create the API catch-all route

```typescript
// app/api/auth/[...all]/route.ts
import { auth } from '@/lib/auth';
import { toNextJsHandler } from 'better-auth/next-js';

export const { GET, POST } = toNextJsHandler(auth);
```

### 8. Create the auth client (for client components)

```typescript
// lib/auth-client.ts
import { createAuthClient } from 'better-auth/react';

export const authClient = createAuthClient({
  baseURL: process.env.NEXT_PUBLIC_APP_URL ?? 'http://localhost:3000',
});

export const {
  signIn,
  signOut,
  signUp,
  useSession,
  requestPasswordReset, // NOT `forgetPassword` — that name does not exist on the client
  resetPassword,
} = authClient;
```

> **Password reset method name:** the client method is **`requestPasswordReset`**, not `forgetPassword`. Calling `authClient.forgetPassword(...)` throws "not a function." To kick off a reset: `await requestPasswordReset({ email, redirectTo: '/reset-password' })`, then on the reset page call `await resetPassword({ newPassword, token })`.

### 9. Add session middleware for protected routes

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

### 10. Verify session server-side in API routes

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

### 11. Verify session in Server Components

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

## Sign-up Page (with email verification)

If you enabled `requireEmailVerification: true` in `lib/auth.ts`, the naive "redirect to /dashboard on sign-up" flow **breaks** — Better Auth does not create a session until the email is verified, so the redirect lands the user back on the sign-in page with no explanation. Instead, render a "check your email" state. Paste this as-is:

```typescript
// app/sign-up/page.tsx
'use client';
import { signUp } from '@/lib/auth-client';
import { useState } from 'react';

export default function SignUpPage() {
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [name, setName] = useState('');
  const [sent, setSent] = useState(false);
  const [error, setError] = useState('');

  async function handleSubmit(e: React.FormEvent) {
    e.preventDefault();
    setError('');
    const { error } = await signUp.email({ email, password, name });
    if (error) {
      setError(error.message ?? 'Sign-up failed');
      return;
    }
    // With requireEmailVerification, there is no session yet — show the check-email state
    setSent(true);
  }

  if (sent) {
    return (
      <div>
        <h1>Check your email</h1>
        <p>We sent a verification link to <strong>{email}</strong>. Click it to finish creating your account, then sign in.</p>
      </div>
    );
  }

  return (
    <form onSubmit={handleSubmit}>
      <input value={name} onChange={e => setName(e.target.value)} placeholder="Name" />
      <input type="email" value={email} onChange={e => setEmail(e.target.value)} placeholder="Email" />
      <input type="password" value={password} onChange={e => setPassword(e.target.value)} placeholder="Password" />
      {error && <p role="alert">{error}</p>}
      <button type="submit">Create account</button>
    </form>
  );
}
```

> If you did **not** enable `requireEmailVerification`, sign-up returns a session immediately and you can redirect to `/dashboard` instead of showing the check-email state.

### Sending the verification email (Resend)

Wire `emailVerification.sendVerificationEmail` in `lib/auth.ts`:

```typescript
emailVerification: {
  sendOnSignUp: true,
  async sendVerificationEmail({ user, url }) {
    await resend.emails.send({
      from: 'onboarding@resend.dev', // or your verified domain
      to: user.email,
      subject: 'Verify your email',
      html: `<a href="${url}">Verify your email</a>`,
    });
  },
},
```

> ⚠️ **Resend test mode delivers to ONE address only.** Until you verify a custom domain in Resend, the API only delivers to the email address of the Resend account owner — and only when sending from `onboarding@resend.dev`. Emails to any other recipient silently fail (they show as "delivered" in some clients but never arrive). **Tell the user this before they test**: have them sign up with the *same email as their Resend account*, or verify a domain first. This single fact accounts for most "the verification email never came" reports.

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

## Database tables

With the Drizzle adapter the tables are **not** auto-created — you define them in `db/auth-schema.ts` (step 5) and push them with `npm run db:push`. The four tables:
- `user` — name, email, emailVerified, image, createdAt, updatedAt
- `session` — token, userId, expiresAt, ipAddress, userAgent
- `account` — accountId, providerId, userId, password (links OAuth + stores password hash)
- `verification` — for email-verification and password-reset tokens

You can extend the `user` table by adding columns in `db/auth-schema.ts` and telling Better Auth about them via `additionalFields`.

---

## Environment variables for production (Vercel or Railway)

Add these where your deploy platform stores env vars — **Vercel: Project Settings → Environment Variables**, or **Railway: service → Variables tab**:

```
BETTER_AUTH_SECRET     → your generated secret (same value as local)
BETTER_AUTH_URL        → https://yourdomain.com (your production URL)
AUTH_GOOGLE_CLIENT_ID  → your Google client ID
AUTH_GOOGLE_CLIENT_SECRET → your Google client secret
```

**Important:** Update your Google OAuth redirect URIs in Google Cloud Console to include your production URL: `https://yourdomain.com/api/auth/callback/google`
