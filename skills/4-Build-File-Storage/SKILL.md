---
name: add-files
description: MUST BE USED when the user wants to add or extend file uploads, file sharing, folders, tags, image transforms, or per-user quotas. Detects existing storage (AWS S3, Firebase Storage, R2, UploadThing) and extends it; if none, scaffolds AWS S3 + CloudFront CDN. Always enforces server-side validation, ownership checks, and magic-byte MIME verification.
when_to_use: "User says 'add file uploads', 'let users upload files', 'add image uploads', 'set up S3', 'add file storage'."
---

# /add-files

You add file features. The default is **AWS S3 + CloudFront CDN**. If a different provider is already wired, adapt to it — never migrate.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Server-side validation (file type, size, ownership) is non-negotiable — never rely on client-side checks alone.
- Confirm storage bucket permissions (public vs. private) with the user before wiring. **S3 buckets must be private by default** — CloudFront handles delivery.
- Check storage quotas and pricing implications before enabling large-file uploads.

## Procedure

### Step 0: Check prerequisites (beginners — read this first)

If the user is setting up file storage for the first time, ask:

> Before we write any code, let's confirm your AWS setup. Answer yes/no to each:
>
> 1. **Do you have an AWS account?** (Sign up free at aws.amazon.com — credit card required but free tier covers most dev usage)
> 2. **Do you have an S3 bucket created?** (We'll need a bucket name and region)
> 3. **Do you have an IAM user with S3 access keys?** (Access Key ID + Secret Access Key)
> 4. **Do you have a CloudFront distribution pointing at your S3 bucket?** (This is what serves files to users)
>
> If any of these are "no," tell me and I'll walk you through the setup step by step before we write any code.

Wait for the user. Walk through any missing setup before proceeding.

### Step 0b: AWS setup walkthrough (if needed)

**Creating the S3 bucket:**
1. Sign in to [console.aws.amazon.com](https://console.aws.amazon.com)
2. Search for "S3" → click "Create bucket"
3. Choose a name (e.g. `myapp-uploads`) and a region close to your users (e.g. `us-east-1`)
4. **Block all public access** — leave this checked (CloudFront handles delivery)
5. Click Create bucket

**Creating an IAM user with S3 access:**
1. Go to IAM → Users → Create user
2. Name: `myapp-s3-user`
3. Attach policy: click "Attach policies directly" → create inline policy with this JSON:
   ```json
   {
     "Version": "2012-10-17",
     "Statement": [{
       "Effect": "Allow",
       "Action": ["s3:GetObject", "s3:PutObject", "s3:DeleteObject"],
       "Resource": "arn:aws:s3:::YOUR-BUCKET-NAME/*"
     }]
   }
   ```
4. Create user → go to Security credentials tab → Create access key → Application running outside AWS
5. Copy the **Access Key ID** and **Secret Access Key** — you only see the secret once

**Creating a CloudFront distribution:**
1. Go to CloudFront → Create distribution
2. Origin domain: select your S3 bucket
3. Origin access: choose "Origin access control settings (recommended)" → create new OAC
4. Copy the bucket policy that CloudFront shows you and apply it to your S3 bucket
5. Default cache behavior: Viewer protocol policy = Redirect HTTP to HTTPS
6. Create distribution → copy the **Distribution domain name** (e.g. `d1234abcd.cloudfront.net`)
7. For signed URLs (private files): go to Key Management → Public Keys → create a key pair; note the Key Pair ID and download the private key

### Step 1: Detect

In parallel:
- `stack-detector` — what storage (if any) is detected
- `pattern-finder` — "Find existing file upload/download code: route, ownership check, size/type validation"

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| Nothing | Install AWS S3 + CloudFront (default) |
| S3 wired (`@aws-sdk/client-s3`) | Extend with S3 |
| R2 wired (`@aws-sdk/client-s3` pointed at R2 endpoint) | Extend with R2 |
| UploadThing wired | Extend with UploadThing |
| Firebase Storage wired | Extend Firebase Storage; do not migrate |

### Step 3: Ask what to add

> What file feature do you need?
> 1. **Basic upload** (no file handling exists yet)
> 2. **File sharing** (generate share links, optional expiry)
> 3. **Folders / nesting**
> 4. **Tags / metadata**
> 5. **Search** (by name, tag, content)
> 6. **Image transformations** (resize, format conversion)
> 7. **Versioning** (keep history)
> 8. **Quotas / plan-based limits**

Then specifics:
- Max file size
- Allowed MIME types (images only? PDFs? any?)
- Per-user storage quota (e.g., free 100MB, pro 10GB)
- Public vs private files
- Direct upload to S3 (pre-signed URL) vs through your server

### Step 4: Plan

Always include:
- DB table for file metadata (id, userId, s3Key, name, mime, size, uploadedAt)
- Server-side validation (size, MIME via magic bytes — not just extension)
- Ownership check on every read/write
- CloudFront signed URLs for private file delivery (or public CloudFront URLs for public files)
- S3 bucket policy restricting cross-user access

### Step 5: Execute

Write code, mirroring existing patterns.

### Step 6: Verify

- Upload a file → confirm it appears in the S3 bucket (check AWS Console)
- Fetch the file via a CloudFront URL → confirm it loads
- Try to access another user's file directly → confirm 403
- Try to upload a file larger than the limit → confirm rejected
- Try to upload a disallowed MIME type → confirm rejected (test by renaming `.exe` to `.jpg` to verify magic-byte check)
- Confirm file metadata in DB matches what's in S3
- Test delete: removing DB row also removes S3 object (or schedules cleanup)

---

## AWS S3 + CloudFront Patterns (Primary)

### Environment variables

```
AWS_ACCESS_KEY_ID=
AWS_SECRET_ACCESS_KEY=
AWS_REGION=us-east-1
AWS_S3_BUCKET=myapp-uploads
CLOUDFRONT_DOMAIN=d1234abcd.cloudfront.net
CLOUDFRONT_KEY_PAIR_ID=APKA...          # Only needed for signed (private) URLs
CLOUDFRONT_PRIVATE_KEY=-----BEGIN RSA PRIVATE KEY-----\n...  # Only needed for signed URLs
```

### t3-env validation

```typescript
// src/env.ts (add these to your existing env.ts)
server: {
  AWS_ACCESS_KEY_ID: z.string().min(1),
  AWS_SECRET_ACCESS_KEY: z.string().min(1),
  AWS_REGION: z.string().min(1),
  AWS_S3_BUCKET: z.string().min(1),
  CLOUDFRONT_DOMAIN: z.string().min(1),
  // Only add these if using private/signed URLs:
  CLOUDFRONT_KEY_PAIR_ID: z.string().optional(),
  CLOUDFRONT_PRIVATE_KEY: z.string().optional(),
}
```

### S3 client setup

```typescript
// lib/s3.ts
import { S3Client } from '@aws-sdk/client-s3';
import { env } from '@/env';

export const s3 = new S3Client({
  region: env.AWS_REGION,
  credentials: {
    accessKeyId: env.AWS_ACCESS_KEY_ID,
    secretAccessKey: env.AWS_SECRET_ACCESS_KEY,
  },
});
```

### Pre-signed upload URL (recommended — client uploads directly to S3)

```typescript
// app/api/files/presign/route.ts
import { PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';
import { s3 } from '@/lib/s3';
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';
import { env } from '@/env';

const MAX_SIZE = 10 * 1024 * 1024; // 10MB
const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];

export async function POST(req: Request) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) return new Response('Unauthorized', { status: 401 });

  const { fileName, mime, size } = await req.json();
  if (size > MAX_SIZE) return new Response('File too large', { status: 413 });
  if (!ALLOWED_MIME.includes(mime)) return new Response('File type not allowed', { status: 400 });

  // Each user's files are namespaced under their user ID
  const key = `users/${session.user.id}/${Date.now()}-${fileName}`;

  const url = await getSignedUrl(
    s3,
    new PutObjectCommand({
      Bucket: env.AWS_S3_BUCKET,
      Key: key,
      ContentType: mime,
      ContentLength: size,
    }),
    { expiresIn: 600 }, // 10 minutes to complete the upload
  );

  return Response.json({ uploadUrl: url, key });
}
```

After the client uploads directly to S3, it calls your API to record the metadata:

```typescript
// app/api/files/route.ts (POST = record metadata after upload)
import { db } from '@/db';
import { files } from '@/db/schema';
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';

export async function POST(req: Request) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) return new Response('Unauthorized', { status: 401 });

  const { key, name, mime, size } = await req.json();

  // Verify the key belongs to this user (prevents spoofing)
  if (!key.startsWith(`users/${session.user.id}/`)) {
    return new Response('Forbidden', { status: 403 });
  }

  const [row] = await db.insert(files).values({
    userId: session.user.id,
    s3Key: key,
    name,
    mime,
    size,
  }).returning();

  return Response.json({ id: row.id });
}
```

### CloudFront delivery URL (for public files)

```typescript
// lib/cloudfront.ts
import { env } from '@/env';

export function getPublicFileUrl(s3Key: string): string {
  return `https://${env.CLOUDFRONT_DOMAIN}/${s3Key}`;
}
```

### CloudFront signed URL (for private files)

```typescript
// lib/cloudfront.ts
import { getSignedUrl } from '@aws-sdk/cloudfront-signer';
import { env } from '@/env';

export function getPrivateFileUrl(s3Key: string, expiresInSeconds = 3600): string {
  return getSignedUrl({
    url: `https://${env.CLOUDFRONT_DOMAIN}/${s3Key}`,
    keyPairId: env.CLOUDFRONT_KEY_PAIR_ID!,
    privateKey: env.CLOUDFRONT_PRIVATE_KEY!,
    dateLessThan: new Date(Date.now() + expiresInSeconds * 1000).toISOString(),
  });
}
```

### File download route (returns a short-lived signed URL)

```typescript
// app/api/files/[id]/route.ts
import { db } from '@/db';
import { files } from '@/db/schema';
import { eq } from 'drizzle-orm';
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';
import { getPrivateFileUrl } from '@/lib/cloudfront';

export async function GET(req: Request, { params }: { params: { id: string } }) {
  const session = await auth.api.getSession({ headers: await headers() });
  if (!session) return new Response('Unauthorized', { status: 401 });

  const [file] = await db.select().from(files).where(eq(files.id, params.id));
  if (!file) return new Response('Not found', { status: 404 });

  // Ownership check — never skip this
  if (file.userId !== session.user.id) return new Response('Forbidden', { status: 403 });

  const url = getPrivateFileUrl(file.s3Key, 3600);
  return Response.json({ url });
}
```

### DB schema for file metadata

```typescript
// db/schema.ts (add this table)
import { pgTable, text, integer, timestamp } from 'drizzle-orm/pg-core';

export const files = pgTable('files', {
  id: text('id').primaryKey().$defaultFn(() => crypto.randomUUID()),
  userId: text('user_id').notNull(),
  s3Key: text('s3_key').notNull(),
  name: text('name').notNull(),
  mime: text('mime').notNull(),
  size: integer('size').notNull(),
  uploadedAt: timestamp('uploaded_at').defaultNow().notNull(),
});
```

### Magic-byte verification (run server-side before saving)

```typescript
// lib/file-magic.ts
const SIGNATURES: Record<string, number[][]> = {
  'image/jpeg': [[0xFF, 0xD8, 0xFF]],
  'image/png': [[0x89, 0x50, 0x4E, 0x47]],
  'image/webp': [[0x52, 0x49, 0x46, 0x46]],
  'application/pdf': [[0x25, 0x50, 0x44, 0x46]],
};

export function verifyMagicBytes(buffer: Buffer, claimedMime: string): boolean {
  const sigs = SIGNATURES[claimedMime];
  if (!sigs) return false;
  return sigs.some(sig => sig.every((byte, i) => buffer[i] === byte));
}
```

---

## UploadThing Patterns (if detected)

```typescript
// app/api/uploadthing/core.ts
import { createUploadthing, type FileRouter } from 'uploadthing/next';
import { auth } from '@/lib/auth';
import { headers } from 'next/headers';
import { db } from '@/db';
import { files } from '@/db/schema';

const f = createUploadthing();

export const ourFileRouter = {
  imageUploader: f({ image: { maxFileSize: '4MB' } })
    .middleware(async () => {
      const session = await auth.api.getSession({ headers: await headers() });
      if (!session) throw new Error('Unauthorized');
      return { userId: session.user.id };
    })
    .onUploadComplete(async ({ metadata, file }) => {
      await db.insert(files).values({
        userId: metadata.userId,
        s3Key: file.key,
        name: file.name,
        mime: file.type,
        size: file.size,
      });
    }),
} satisfies FileRouter;
```

---

## Firebase Storage Patterns (for existing projects only)

If the project already uses Firebase Storage, extend it — do not migrate. See the Firebase Storage implementation in the legacy pattern block below.

```typescript
// lib/firebase-storage.ts (existing Firebase projects only)
import { getStorage } from 'firebase-admin/storage';
import { adminApp } from './firebase-admin';

const bucket = getStorage(adminApp).bucket();

export async function uploadFile(userId: string, fileName: string, buffer: Buffer, mime: string) {
  const path = `users/${userId}/${Date.now()}-${fileName}`;
  const file = bucket.file(path);
  await file.save(buffer, { metadata: { contentType: mime } });
  return path;
}

export async function getSignedDownloadUrl(path: string, expiresInSeconds = 3600) {
  const [url] = await bucket.file(path).getSignedUrl({
    action: 'read',
    expires: Date.now() + expiresInSeconds * 1000,
  });
  return url;
}
```

---

## Common Security Pitfalls

- **Trusting MIME type from the client** — always verify with magic bytes server-side
- **Trusting filename** — strip path separators (`/`, `\`, `..`) to prevent path traversal
- **No size limit** — easy DoS / cost vector; always validate before accepting the upload
- **No quota** — a single user could upload gigabytes; enforce per-user limits
- **Public S3 bucket** — S3 buckets must be private; use CloudFront for delivery
- **Long-lived signed URLs** — keep expiry < 1 hour for downloads, < 10 min for uploads
- **Storing PII in filenames** — file paths appear in logs; use UUIDs or hashes for S3 keys

## Rules

- **Server-side validation always** — client validation is for UX only.
- **Magic-byte verification** for any file that will be served back to users.
- **Per-user S3 paths** — `users/{userId}/...` pattern; ownership check on every read.
- **S3 bucket is private** — CloudFront is the only delivery mechanism.
- **Quotas enforced before upload** — don't accept the file then reject.
- **Cleanup orphans** — when DB row is deleted, also delete the S3 object.

## If Something Goes Wrong

- **Upload fails with CORS error** — in the S3 bucket settings, add a CORS configuration allowing your app's origin for PUT requests; add `localhost` for dev and your production domain for prod.
- **CloudFront returns 403** — confirm the bucket policy was updated to allow CloudFront OAC access (AWS Console shows this policy during CloudFront setup; you must apply it to the bucket manually).
- **Pre-signed URL expired** — the client has 10 minutes to complete the upload; if the upload is slow, increase `expiresIn` in the presigner.
- **File validation rejecting valid files** — check magic-byte MIME detection; browser-reported MIME types can be spoofed.
- **Ownership check fails for valid user** — confirm the `userId` field in the DB record matches the auth session's `session.user.id` format exactly (string vs UUID mismatch is common).
- **IAM credentials not working** — verify the IAM policy is attached to the correct user and targets the correct bucket ARN; test with the AWS CLI: `aws s3 ls s3://your-bucket-name`.
