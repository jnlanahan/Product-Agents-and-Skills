---
title: "Vitest Patterns"
skill: "4-Build-Tests"
---

# Vitest Patterns (Next.js / Vite)

## Install

```bash
npm install --save-dev vitest @vitest/coverage-v8 @testing-library/react @testing-library/jest-dom @testing-library/user-event jsdom
```

## Config

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

## Setup file

```typescript
// src/test/setup.ts
import '@testing-library/jest-dom';
```

## package.json scripts

```json
{
  "scripts": {
    "test": "vitest run",
    "test:watch": "vitest",
    "test:coverage": "vitest run --coverage"
  }
}
```

## Unit test (pure function)

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

## Component test

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

## API route test (Next.js App Router)

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
