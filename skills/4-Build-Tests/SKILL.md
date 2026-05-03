---
name: 4-Build-Tests
description: MUST BE USED when a project has no test framework or when adding a first meaningful test suite. Scaffolds Vitest (Next.js/Vite) or Jest (Express/Node), adds test utilities and setup, and writes the first passing unit + integration tests following the project's existing patterns. Trigger on `/setup-tests`, "add tests", "write tests", "set up testing", "add Vitest", "add Jest", "add testing infrastructure".
---

# /setup-tests

You add a test framework and a first meaningful test suite to a project that has none or almost none.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — framework (Next.js, Express, Vite), existing test setup (jest, vitest, mocha)
- `pattern-finder` — "Find existing tests: *.test.ts, *.spec.ts, __tests__/ directories"

Read `_stack-preferences.md`.

### Step 2: Determine mode

| Detected framework | Test framework | Action |
|---|---|---|
| Next.js, no tests | Vitest + Testing Library | Scaffold fresh |
| Vite app, no tests | Vitest + Testing Library | Scaffold fresh |
| Express / Node, no tests | Vitest or Jest | Scaffold fresh |
| Tests already exist | Whatever's there | Extend — don't reinstall; add missing coverage |

### Step 3: Confirm scope

> What should the first test suite cover?
> 1. **Unit tests** — pure functions, utilities, business logic
> 2. **Component tests** — React components (Testing Library)
> 3. **API / route tests** — HTTP endpoints
> 4. **Integration tests** — real DB queries (requires a test database)

### Step 4: Execute

Scaffold per the patterns below. Write at least 3 tests that actually pass before marking done.

### Step 5: Verify

- `npm test` passes with no failures
- At least 3 meaningful assertions in the suite
- Tests run in under 10 seconds (for unit/component tests)

---

## Vitest Patterns (Next.js / Vite)

### Install

```bash
npm install --save-dev vitest @vitest/coverage-v8 @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

### Config

```typescript
// vitest.config.ts
import { defineConfig } from 'vitest/config';
import react from '@vitejs/plugin-react';
import path from 'path';

export default defineConfig({
  plugins: [react()],
  test: {
    environment: 'jsdom',
    globals: true,
    setupFiles: ['./src/test/setup.ts'],
    coverage: { provider: 'v8', reporter: ['text', 'html'] },
  },
  resolve: {
    alias: { '@': path.resolve(__dirname, './src') },
  },
});
```

### Setup file

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom';
```

### package.json scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

### Unit test (pure function)

```typescript
// src/lib/__tests__/formatters.test.ts
import { describe, it, expect } from 'vitest';
import { formatCurrency } from '../formatters';

describe('formatCurrency', () => {
  it('formats cents as dollars', () => {
    expect(formatCurrency(1000)).toBe('$10.00');
  });
  it('handles zero', () => {
    expect(formatCurrency(0)).toBe('$0.00');
  });
  it('handles negative values', () => {
    expect(formatCurrency(-500)).toBe('-$5.00');
  });
});
```

### Component test

```tsx
// src/components/__tests__/Button.test.tsx
import { render, screen } from '@testing-library/react';
import userEvent from '@testing-library/user-event';
import { describe, it, expect, vi } from 'vitest';
import { Button } from '../Button';

describe('Button', () => {
  it('renders the label', () => {
    render(<Button onClick={vi.fn()}>Submit</Button>);
    expect(screen.getByRole('button', { name: 'Submit' })).toBeInTheDocument();
  });

  it('calls onClick when clicked', async () => {
    const onClick = vi.fn();
    render(<Button onClick={onClick}>Submit</Button>);
    await userEvent.click(screen.getByRole('button'));
    expect(onClick).toHaveBeenCalledOnce();
  });

  it('is disabled when the disabled prop is true', () => {
    render(<Button onClick={vi.fn()} disabled>Submit</Button>);
    expect(screen.getByRole('button')).toBeDisabled();
  });
});
```

### API route test (Next.js App Router)

```typescript
// src/app/api/__tests__/health.test.ts
import { describe, it, expect } from 'vitest';
import { GET } from '../health/route';

describe('GET /api/health', () => {
  it('returns 200 with ok status', async () => {
    const response = await GET(new Request('http://localhost/api/health'));
    expect(response.status).toBe(200);
    const body = await response.json();
    expect(body.status).toBe('ok');
  });
});
```

---

## Jest Patterns (Express / Node)

### Install

```bash
npm install --save-dev jest @types/jest ts-jest supertest @types/supertest
```

### Config

```javascript
// jest.config.js
module.exports = {
  preset: 'ts-jest',
  testEnvironment: 'node',
  testMatch: ['**/__tests__/**/*.test.ts'],
  collectCoverageFrom: ['src/**/*.ts', '!src/**/*.d.ts'],
};
```

### package.json scripts

```json
{
  "scripts": {
    "test": "jest",
    "test:watch": "jest --watch",
    "test:coverage": "jest --coverage"
  }
}
```

### API test (Express + supertest)

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

## Integration Tests (Real DB — Drizzle + Neon)

Only scaffold these when the user confirms a test database is available. Never use the production DB.

### Setup

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

---

## Rules

- **Tests verify behavior through public interfaces** — not internals. Test what the function/component/route does, not how.
- **Mock at system boundaries** — DB, third-party APIs, time (`vi.useFakeTimers()`), randomness. Never mock internal collaborators.
- **No production DB in tests** — integration tests must use `DATABASE_URL_TEST`.
- **At least 3 passing tests before marking done** — proves setup works AND provides real coverage.
- **Test file naming**: `*.test.ts` colocated with source, or in `__tests__/` directories adjacent to source.
- **Don't test TypeScript types** — `tsc --noEmit` handles that; tests verify runtime behavior.
