---
name: setup-tests
description: MUST BE USED when a project has no test framework or when adding a first meaningful test suite. Scaffolds Vitest (Next.js/Vite) or Jest (Express/Node), adds test utilities and setup, and writes the first passing unit + integration tests following the project's existing patterns.
when_to_use: "User says 'add tests', 'write tests', 'set up testing', 'add Vitest', 'add Jest', 'scaffold the test suite', 'I have no tests'."
---

# /setup-tests

You add a test framework and a first meaningful test suite to a project that has none or almost none.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Run `stack-detector` first if framework choice is unclear — Vitest for Vite/Next.js, Jest for plain Node/Express; mixing them causes config conflicts.
- Do not install both Vitest and Jest in the same project; pick one and wire it consistently.
- The first test suite must pass on a clean install — flaky setup tests are worse than no tests.

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

→ See [vitest-patterns.md](references/vitest-patterns.md) for Vitest config, setup file, npm scripts, unit test, component test, and API route test examples.

→ See [jest-patterns.md](references/jest-patterns.md) for Jest config, npm scripts, Express/supertest API test, and integration test examples with a real test DB.

## Rules

- **Tests verify behavior through public interfaces** — not internals. Test what the function/component/route does, not how.
- **Mock at system boundaries** — DB, third-party APIs, time (`vi.useFakeTimers()`), randomness. Never mock internal collaborators.
- **No production DB in tests** — integration tests must use `DATABASE_URL_TEST`.
- **At least 3 passing tests before marking done** — proves setup works AND provides real coverage.
- **Test file naming**: `*.test.ts` colocated with source, or in `__tests__/` directories adjacent to source.
- **Don't test TypeScript types** — `tsc --noEmit` handles that; tests verify runtime behavior.

## If Something Goes Wrong

- **Test runner fails to start** — check for a conflicting `jest.config.*` or `vitest.config.*`; delete any duplicate config files and restart.
- **Tests pass in isolation but fail together** — a shared global state or singleton is leaking between tests; add `beforeEach`/`afterEach` cleanup or use `vi.resetAllMocks()`.
- **Coverage not collected** — confirm the `coverage` provider is set in the config (`istanbul` or `v8` for Vitest); run with `--coverage` flag explicitly.
- **TypeScript type errors in test files** — add the test framework types to `tsconfig.json` under `compilerOptions.types` (`vitest/globals` or `jest`).