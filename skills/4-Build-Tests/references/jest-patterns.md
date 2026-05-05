---
title: "Jest Patterns"
skill: "4-Build-Tests"
---

# Jest Patterns (Express / Node)

## Install

```bash
npm install --save-dev jest @types/jest ts-jest supertest @types/supertest
```

## Config

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.ts'],
  collectCoverageFrom: ['src/**/*.ts', '!src/**/*.d.ts'],
};
```

## package.json scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

## API test (Express + supertest)

```typescript
// src/__tests__/health.test.ts
import request from 'supertest';
import { app } from '../app';

describe('GET /health', () => {
  it('returns 200 OK', async () => {
    const res = await request(app).get('/health');
    expect(res.status).toBe(200);
    expect(res.body.status).toBe('ok');
  });
});
```

---

# Integration Tests (Real DB — Drizzle + Neon)

Only scaffold these when the user confirms a test database is available. Never use the production DB.

## Setup

```
DATABASE_URL_TEST=postgresql://...  (in .env.test or CI secrets)
```

```typescript
// src/test/db.ts
import { drizzle } from 'drizzle-orm/neon-http';
import { neon } from '@neondatabase/serverless';

const sql = neon(process.env.DATABASE_URL_TEST!);
export const testDb = drizzle(sql);
```

```typescript
// src/__tests__/users.integration.test.ts
import { testDb } from '../test/db';
import { users } from '../db/schema';
import { beforeEach, afterAll, describe, it, expect } from 'vitest';

beforeEach(async () => {
  await testDb.delete(users);
});

afterAll(async () => {
  await testDb.delete(users);
});

describe('user creation', () => {
  it('inserts and returns a user', async () => {
    const [user] = await testDb.insert(users).values({ email: 'test@example.com' }).returning();
    expect(user.email).toBe('test@example.com');
  });
});
```
