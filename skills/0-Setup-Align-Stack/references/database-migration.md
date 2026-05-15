# Database Migration Reference — Any DB/ORM → Neon Postgres + Drizzle

Referenced by: [../SKILL.md](../SKILL.md) (Database wave and ORM wave)

Three scenarios are covered:

1. **Supabase Postgres → Neon Postgres** (same dialect, different host)
2. **PlanetScale (MySQL) → Neon Postgres** (dialect change; schema rewrite required)
3. **Prisma / TypeORM → Drizzle** (ORM swap; DB host may or may not change)

---

## Scenario A: Supabase Postgres → Neon Postgres

Supabase and Neon are both Postgres — this is a host migration, not a dialect change.

### Connection setup (Drizzle + Neon)

```ts
// db/index.ts
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';
import * as schema from './schema';

const sql = neon(process.env.DATABASE_URL!);
export const db = drizzle(sql, { schema });
```

Required env var: `DATABASE_URL=postgresql://...@...neon.tech/neondb?sslmode=require`

### Translating Supabase client queries to Drizzle

The Supabase JS client wraps Postgres in a REST-like API. Each call maps to a Drizzle equivalent:

| Supabase client | Drizzle |
|---|---|
| `supabase.from('users').select('*')` | `db.select().from(users)` |
| `supabase.from('users').select('id, email')` | `db.select({ id: users.id, email: users.email }).from(users)` |
| `supabase.from('users').insert({ ... })` | `db.insert(users).values({ ... }).returning()` |
| `supabase.from('users').update({ ... }).eq('id', id)` | `db.update(users).set({ ... }).where(eq(users.id, id))` |
| `supabase.from('users').delete().eq('id', id)` | `db.delete(users).where(eq(users.id, id))` |
| `.select().eq('status', 'active').order('created_at', { ascending: false })` | `.select().from(users).where(eq(users.status, 'active')).orderBy(desc(users.createdAt))` |

### Translating Supabase schema types to Drizzle columns

| Supabase / Postgres type | Drizzle column |
|---|---|
| `uuid` (default primary key) | `uuid('id').defaultRandom().primaryKey()` |
| `text` | `text('field')` |
| `boolean` | `boolean('field')` |
| `timestamp with time zone` | `timestamp('created_at', { withTimezone: true }).defaultNow()` |
| `jsonb` | `jsonb('data')` |
| `integer` | `integer('field')` |
| `bigint` | `bigint('field', { mode: 'number' })` |
| `numeric` / `decimal` | `numeric('amount', { precision: 10, scale: 2 })` |

Example schema (`db/schema.ts`):

```ts
import { pgTable, uuid, text, timestamp, boolean, jsonb } from 'drizzle-orm/pg-core';

export const users = pgTable('users', {
  id: uuid('id').defaultRandom().primaryKey(),
  email: text('email').notNull().unique(),
  name: text('name'),
  emailVerified: boolean('email_verified').default(false),
  metadata: jsonb('metadata'),
  createdAt: timestamp('created_at', { withTimezone: true }).defaultNow().notNull(),
});
```

### Data migration (when Supabase has real data)

**Step 1: Export from Supabase**

Option A — pg_dump (best for complete exports):
```bash
pg_dump \
  -h db.<project-ref>.supabase.co \
  -p 5432 \
  -U postgres \
  -d postgres \
  --data-only \
  --no-owner \
  --no-acl \
  -F c \
  -f supabase-backup.dump
```
Password: from Supabase Dashboard → Settings → Database → Database password

Option B — Supabase CLI:
```bash
supabase db dump --data-only -f supabase-backup.sql
```

**Step 2: Import to Neon**

```bash
pg_restore \
  -d "postgresql://<user>:<password>@<host>.neon.tech/neondb?sslmode=require" \
  --data-only \
  --no-owner \
  --no-acl \
  supabase-backup.dump
```

**Step 3: Verify row counts**

Run this on both databases and confirm the numbers match:
```sql
SELECT relname AS table_name, n_live_tup AS row_count
FROM pg_stat_user_tables
WHERE schemaname = 'public'
ORDER BY relname;
```

**Important:** Supabase adds system tables (`auth.users`, `storage.objects`, etc.) and Row Level Security policies that do NOT need to migrate. Better Auth manages user storage in Neon directly; Supabase RLS is replaced by application-level auth checks.

---

## Scenario B: PlanetScale (MySQL) → Neon Postgres

This requires a dialect conversion — MySQL syntax and types differ from Postgres.

### Key type mapping

| MySQL / PlanetScale | Postgres / Neon | Drizzle column |
|---|---|---|
| `TINYINT(1)` | `BOOLEAN` | `boolean('field')` |
| `DATETIME` | `TIMESTAMP WITH TIME ZONE` | `timestamp('field', { withTimezone: true })` |
| `JSON` | `JSONB` | `jsonb('field')` |
| `BIGINT UNSIGNED AUTO_INCREMENT` | `BIGSERIAL` or UUID | `uuid('id').defaultRandom().primaryKey()` |
| `VARCHAR(255)` | `TEXT` | `text('field')` |
| `TINYTEXT` / `MEDIUMTEXT` | `TEXT` | `text('field')` |
| `ENUM('a','b')` | Postgres enum or text + check | `pgEnum('status', ['a','b'])` |
| `DECIMAL(10,2)` | `NUMERIC(10,2)` | `numeric('amount', { precision: 10, scale: 2 })` |

### Query differences to rewrite

| MySQL | Postgres |
|---|---|
| `IF(condition, a, b)` | `CASE WHEN condition THEN a ELSE b END` |
| `GROUP_CONCAT(col)` | `STRING_AGG(col, ',')` |
| `IFNULL(x, y)` | `COALESCE(x, y)` |
| `NOW()` | `NOW()` (same) |
| `LIMIT n OFFSET m` | `LIMIT n OFFSET m` (same) |

PlanetScale uses Vitess which disables foreign keys. When migrating to Postgres, add proper `references()` in your Drizzle schema — this enforces referential integrity you didn't have before.

### Data migration tools for MySQL → Postgres

PlanetScale doesn't support `pg_dump`. Use one of:

- **pgloader** (recommended for large datasets):
  ```
  pgloader mysql://user:pass@host/dbname postgresql://user:pass@host/neondb
  ```
  pgloader handles type coercion automatically.

- **Airbyte** (free tier, GUI-driven): source = MySQL, destination = Postgres

- **Manual CSV export**: for small tables, export each table as CSV from PlanetScale, then `\COPY table FROM 'file.csv' CSV HEADER` in psql against Neon

---

## Scenario C: Prisma / TypeORM → Drizzle

Pure ORM swap. The DB host does not change in this wave (unless combined with Scenario A or B).

### Schema translation: Prisma → Drizzle

**Prisma schema:**
```prisma
model User {
  id        String   @id @default(cuid())
  email     String   @unique
  name      String?
  createdAt DateTime @default(now())
  posts     Post[]
}

model Post {
  id       String  @id @default(cuid())
  title    String
  body     String?
  authorId String
  author   User    @relation(fields: [authorId], references: [id])
}
```

**Drizzle equivalent:**
```ts
import { pgTable, text, timestamp } from 'drizzle-orm/pg-core';
import { relations } from 'drizzle-orm';
import { createId } from '@paralleldrive/cuid2';

export const users = pgTable('User', {
  id: text('id').$defaultFn(() => createId()).primaryKey(),
  email: text('email').notNull().unique(),
  name: text('name'),
  createdAt: timestamp('createdAt').defaultNow().notNull(),
});

export const posts = pgTable('Post', {
  id: text('id').$defaultFn(() => createId()).primaryKey(),
  title: text('title').notNull(),
  body: text('body'),
  authorId: text('authorId').notNull().references(() => users.id),
});

export const usersRelations = relations(users, ({ many }) => ({
  posts: many(posts),
}));

export const postsRelations = relations(posts, ({ one }) => ({
  author: one(users, { fields: [posts.authorId], references: [users.id] }),
}));
```

Keep the same table names Prisma used (`"User"`, `"Post"`) so the existing data rows don't need renaming.

### Query translation: Prisma → Drizzle

| Prisma | Drizzle |
|---|---|
| `prisma.user.findMany()` | `db.select().from(users)` |
| `prisma.user.findUnique({ where: { id } })` | `db.select().from(users).where(eq(users.id, id))` |
| `prisma.user.findMany({ where: { email } })` | `db.select().from(users).where(eq(users.email, email))` |
| `prisma.user.create({ data })` | `db.insert(users).values(data).returning()` |
| `prisma.user.update({ where: { id }, data })` | `db.update(users).set(data).where(eq(users.id, id)).returning()` |
| `prisma.user.delete({ where: { id } })` | `db.delete(users).where(eq(users.id, id))` |
| `prisma.user.count()` | `db.select({ count: count() }).from(users)` |
| `prisma.user.findMany({ include: { posts: true } })` | `db.query.users.findMany({ with: { posts: true } })` |
| `prisma.user.findMany({ orderBy: { createdAt: 'desc' } })` | `db.select().from(users).orderBy(desc(users.createdAt))` |
| `prisma.user.findMany({ take: 10, skip: 20 })` | `db.select().from(users).limit(10).offset(20)` |

### Drizzle relational queries (replaces Prisma `include`)

```ts
// Requires `schema` passed to drizzle() constructor
const usersWithPosts = await db.query.users.findMany({
  with: { posts: true },
});

// Filtered relation
const user = await db.query.users.findFirst({
  where: eq(users.id, userId),
  with: {
    posts: {
      where: eq(posts.published, true),
      orderBy: desc(posts.createdAt),
    },
  },
});
```

### Schema translation: TypeORM → Drizzle

TypeORM uses decorator-based entities. Remove the decorators and rewrite as Drizzle table definitions following the same pattern as Prisma above. Key mappings:

| TypeORM decorator | Drizzle column |
|---|---|
| `@PrimaryGeneratedColumn('uuid')` | `uuid('id').defaultRandom().primaryKey()` |
| `@Column({ type: 'text' })` | `text('field')` |
| `@Column({ nullable: true })` | `text('field')` (no `.notNull()`) |
| `@CreateDateColumn()` | `timestamp('created_at').defaultNow().notNull()` |
| `@ManyToOne(() => User)` | `.references(() => users.id)` on the FK column |

### Removing Prisma

```bash
npm uninstall @prisma/client prisma
rm -rf prisma/
```

Remove from `package.json`:
```json
{
  "scripts": {
    "postinstall": "prisma generate"  // delete this line
  }
}
```

### drizzle.config.ts

```ts
import { defineConfig } from 'drizzle-kit';

export default defineConfig({
  schema: './db/schema.ts',
  out: './drizzle',
  dialect: 'postgresql',
  dbCredentials: { url: process.env.DATABASE_URL! },
});
```

### package.json scripts to add

```json
{
  "scripts": {
    "db:generate": "drizzle-kit generate",
    "db:migrate": "drizzle-kit migrate",
    "db:push": "drizzle-kit push",
    "db:studio": "drizzle-kit studio"
  }
}
```

- `db:generate` — generates a new SQL migration from schema changes (use in dev)
- `db:migrate` — applies pending migrations (use in CI and production)
- `db:push` — pushes schema directly without a migration file (use for rapid prototyping only)
- `db:studio` — opens Drizzle Studio browser UI for inspecting the DB
