---
name: add-files
description: MUST BE USED when the user wants to add or extend file uploads, file sharing, folders, tags, image transforms, or per-user quotas. Detects existing storage (Firebase Storage, S3, R2, UploadThing) and extends it; if none, scaffolds Firebase Storage. Always enforces server-side validation, ownership checks, and magic-byte MIME verification. Trigger on `/add-files`, "add file uploads", "add file storage".
---

# /add-files

You add file features. Preference is Firebase Storage. If a different provider is detected, adapt to it.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Server-side validation (file type, size, ownership) is non-negotiable — never rely on client-side checks alone.
- Confirm storage bucket permissions (public vs. private) with the user before wiring; public buckets can expose sensitive uploads.
- Check storage quotas and pricing implications before enabling large-file uploads.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — what storage (if any) is detected
- `pattern-finder` — "Find existing file upload/download code: route, ownership check, size/type validation"

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected | Action |
|---|---|
| Nothing | Install Firebase Storage (preference) |
| Firebase Storage wired | Extend it |
| S3 wired (`@aws-sdk/client-s3`) | Extend with S3 |
| R2 wired (`@aws-sdk/client-s3` pointed at R2 endpoint) | Extend with R2 |
| UploadThing wired | Extend with UploadThing |

### Step 3: Ask what to add

> What file feature?
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
- Allowed MIME types
- Per-user storage quota (e.g., free 100MB, pro 10GB)
- Public vs private files
- Direct upload to storage (signed URL) vs through your server

### Step 4: Plan

Always include:
- DB table for file metadata (id, userId, path, name, mime, size, uploadedAt)
- Server-side validation (size, MIME via magic bytes not extension)
- Ownership check on every read/write
- Storage rules (Firebase) or bucket policy (S3/R2) restricting cross-user access

### Step 5: Execute

Write code, mirroring existing patterns.

### Step 6: Verify

- Upload a file → confirm it appears in storage
- Try to access another user's file → confirm 403
- Try to upload a file > max size → confirm rejected
- Try to upload a disallowed MIME type → confirm rejected (test by renaming `.exe` to `.jpg` to verify magic-byte check)
- Confirm file metadata in DB matches storage state
- Test delete: removing DB row also removes storage object (or schedule for cleanup)

## Firebase Storage Patterns

### Server upload (admin SDK)

```typescript
// lib/firebase-storage.ts
import { getStorage } from 'firebase-admin/storage';
import { adminApp } from './firebase-admin';

const bucket = getStorage(adminApp).bucket();

export async function uploadFile(userId: string, fileName: string, buffer: Buffer, mime: string) {
  const path = `users/${userId}/${Date.now()}-${fileName}`;
  const file = bucket.file(path);
  await file.save(buffer, {
    metadata: { contentType: mime },
  });
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

### Upload route (Next.js with Multer-like form handling)

```typescript
// app/api/files/route.ts
import { verifyIdToken } from '@/lib/auth';
import { uploadFile } from '@/lib/firebase-storage';
import { db } from '@/db';
import { files } from '@/db/schema';

const MAX_SIZE = 10 * 1024 * 1024; // 10MB
const ALLOWED_MIME = ['image/jpeg', 'image/png', 'image/webp', 'application/pdf'];

export async function POST(req: Request) {
  const user = await verifyIdToken(req);
  const formData = await req.formData();
  const file = formData.get('file') as File;

  if (!file) return new Response('No file', { status: 400 });
  if (file.size > MAX_SIZE) return new Response('Too large', { status: 413 });
  if (!ALLOWED_MIME.includes(file.type)) return new Response('Wrong type', { status: 400 });

  // Magic-byte verification
  const buffer = Buffer.from(await file.arrayBuffer());
  if (!verifyMagicBytes(buffer, file.type)) {
    return new Response('Invalid file content', { status: 400 });
  }

  // Quota check
  const used = await getUserStorageUsed(user.uid);
  const quota = await getUserQuota(user.uid);
  if (used + file.size > quota) {
    return new Response('Quota exceeded', { status: 413 });
  }

  const path = await uploadFile(user.uid, file.name, buffer, file.type);
  const [row] = await db.insert(files).values({
    userId: user.uid,
    path,
    name: file.name,
    mime: file.type,
    size: file.size,
  }).returning();

  return Response.json({ id: row.id });
}
```

### Firebase Storage security rules

```
// firebase-storage.rules
rules_version = '2';
service firebase.storage {
  match /b/{bucket}/o {
    match /users/{userId}/{allPaths=**} {
      allow read, write: if request.auth != null && request.auth.uid == userId;
    }
  }
}
```

Deploy with: `firebase deploy --only storage`

### Magic-byte verification

```typescript
// lib/file-magic.ts
const SIGNATURES: Record<string, number[][]> = {
  'image/jpeg': [[0xFF, 0xD8, 0xFF]],
  'image/png': [[0x89, 0x50, 0x4E, 0x47]],
  'image/webp': [[0x52, 0x49, 0x46, 0x46]], // RIFF (then WEBP at offset 8)
  'application/pdf': [[0x25, 0x50, 0x44, 0x46]],
};

export function verifyMagicBytes(buffer: Buffer, claimedMime: string): boolean {
  const sigs = SIGNATURES[claimedMime];
  if (!sigs) return false;
  return sigs.some(sig => sig.every((byte, i) => buffer[i] === byte));
}
```

## S3/R2 Patterns

### Pre-signed URL upload (recommended for large files)

```typescript
// app/api/files/presign/route.ts
import { S3Client, PutObjectCommand } from '@aws-sdk/client-s3';
import { getSignedUrl } from '@aws-sdk/s3-request-presigner';

const s3 = new S3Client({
  region: process.env.AWS_REGION,
  // For R2:
  // endpoint: process.env.R2_ENDPOINT,
  // credentials: { accessKeyId: ..., secretAccessKey: ... },
});

export async function POST(req: Request) {
  const user = await verifyIdToken(req);
  const { fileName, mime } = await req.json();

  const key = `users/${user.uid}/${Date.now()}-${fileName}`;
  const url = await getSignedUrl(
    s3,
    new PutObjectCommand({ Bucket: process.env.S3_BUCKET, Key: key, ContentType: mime }),
    { expiresIn: 600 }, // 10 minutes
  );

  return Response.json({ url, key });
}
```

Client uploads directly to S3 with the signed URL (PUT request). After upload, client confirms to your API which records the metadata.

## UploadThing Patterns

```typescript
// app/api/uploadthing/core.ts
import { createUploadthing, type FileRouter } from 'uploadthing/next';

const f = createUploadthing();

export const ourFileRouter = {
  imageUploader: f({ image: { maxFileSize: '4MB' } })
    .middleware(async ({ req }) => {
      const user = await verifyIdToken(req);
      return { userId: user.uid };
    })
    .onUploadComplete(async ({ metadata, file }) => {
      await db.insert(files).values({
        userId: metadata.userId,
        path: file.url,
        name: file.name,
        mime: file.type,
        size: file.size,
      });
    }),
} satisfies FileRouter;
```

## Common Security Pitfalls

- **Trusting MIME type from the client** — always verify with magic bytes server-side
- **Trusting filename** — strip path separators (`/`, `\`, `..`) to prevent path traversal
- **No size limit** — easy DoS / cost vector
- **No quota** — same
- **Public bucket by default** — both S3 and Firebase need explicit configuration to be private
- **Long-lived signed URLs** — keep expiry < 1 hour for downloads, < 10 min for uploads
- **Storing PII in filenames** — file paths often appear in logs; use UUIDs or hashes for filenames

## Rules

- **Server-side validation always** — client validation is for UX only.
- **Magic-byte verification** for any file that will be served back to users.
- **Per-user storage paths** — `users/{uid}/...` pattern; storage rules enforce isolation.
- **Quotas enforced before upload** — don't accept the file then reject.
- **Cleanup orphans** — when DB row is deleted, also delete the storage object.

## If Something Goes Wrong

- **Upload fails with CORS error** — confirm the storage bucket's CORS policy allows your app's origin; add `localhost` for development and the production domain for production.
- **File validation rejecting valid files** — check magic-byte MIME detection logic; browser-reported MIME types can be spoofed and should not be trusted alone.
- **Storage quota exceeded** — check bucket usage in the provider console; add a user-level quota check before the upload rather than after.
- **Ownership check fails for valid user** — confirm the ownership field in the DB record matches the authenticated user ID format (UUID vs. integer mismatch is common).