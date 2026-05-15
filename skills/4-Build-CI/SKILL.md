---
name: setup-ci
description: MUST BE USED when adding continuous integration to a project that lacks it. Generates GitHub Actions workflows for typecheck, tests, and lint. Vercel auto-deploys from GitHub — no separate deploy CI step needed. Adapts to detected test framework and package manager. Trigger on `/setup-ci`, "add CI", "add GitHub Actions", "automate tests on PR", "wire CI/CD", "deploy on push".
---

# /setup-ci

You add CI to a project that doesn't have it. Preference is GitHub Actions for typecheck + tests. Vercel handles deploys automatically from GitHub — no separate deploy workflow is needed for the default stack.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Vercel deploys automatically on every push to main — you don't need a deploy step in GitHub Actions.
- Show the generated CI config to the user before pushing it; a broken pipeline blocks all PRs.
- Ensure tests pass locally before wiring CI — a red pipeline on day one destroys confidence in the system.

## Procedure

### Step 1: Detect

In parallel:
- `stack-detector` — test framework (Vitest, Jest, Playwright), deploy target (Vercel, Railway, Render, Fly), package manager (npm/pnpm/yarn)
- `pattern-finder` — "Find existing CI files: .github/workflows/, existing npm test/typecheck scripts in package.json"

### Step 2: Determine mode

| Detected | Action |
|---|---|
| No CI config | Scaffold GitHub Actions from scratch |
| `.github/workflows/` exists but incomplete | Extend with missing jobs |
| CircleCI / Jenkins / other | Extend existing; don't add GitHub Actions unless user asks |

### Step 3: Ask scope

> What CI jobs do you want?
> 1. **Typecheck + tests on PR** (recommended minimum — always include this)
> 2. **Lint on PR** (ESLint/Biome)
> 3. **E2E tests (Playwright)** on PR
> 4. **Auto-deploy to Render on merge to main** (only if NOT using Vercel — Vercel auto-deploys from GitHub)
>
> Note: If you're using Vercel, deploys happen automatically on every push — you don't need a deploy job in CI.

### Step 4: Collect secrets

Tell the user which GitHub secrets to add (Settings → Secrets and variables → Actions) before the workflow runs.

### Step 5: Execute

Create `.github/workflows/` files per the patterns below.

### Step 6: Verify

- Push the workflow file to a branch, open a PR
- Watch GitHub → Actions tab — confirm jobs run and pass
- Fix any cache key or missing env var issues before marking done

---

## GitHub Actions Patterns

### Core: typecheck + tests on PR (minimum viable CI)

```yaml
# .github/workflows/ci.yml
name: CI

on:
  pull_request:
    branches: [main]
  push:
    branches: [main]

jobs:
  check:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4

      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'

      - run: npm ci

      - name: Typecheck
        run: npm run typecheck   # or: npx tsc --noEmit

      - name: Test
        run: npm test
        env:
          DATABASE_URL: ${{ secrets.DATABASE_URL_TEST }}
          # Add other env vars your tests require
```

### Add lint job

```yaml
      - name: Lint
        run: npm run lint
```

### pnpm variant

```yaml
      - uses: pnpm/action-setup@v3
        with:
          version: 8
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'pnpm'
      - run: pnpm install --frozen-lockfile
```

### Auto-deploy to Vercel (no CI step needed)

If you're using Vercel, **you do not need a deploy job in GitHub Actions**. Vercel's GitHub integration auto-deploys on every push to main. Just connect your repo in the Vercel dashboard and it handles the rest.

### Auto-deploy to Render on push to main (if not using Vercel)

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - name: Trigger Render deploy
        run: curl -X POST "${{ secrets.RENDER_DEPLOY_HOOK_URL }}"
```

> **Setup**: Render → your service → Settings → Deploy Hooks → copy the URL → add as `RENDER_DEPLOY_HOOK_URL` secret in GitHub.

### Auto-deploy to Railway on push to main

```yaml
# .github/workflows/deploy.yml
name: Deploy

on:
  push:
    branches: [main]

jobs:
  deploy:
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - name: Deploy to Railway
        uses: bervProject/railway-deploy@main
        with:
          railway_token: ${{ secrets.RAILWAY_TOKEN }}
          service: your-service-name
```

> **Setup**: Railway → account settings → Tokens → create token → add as `RAILWAY_TOKEN` in GitHub.

### E2E tests with Playwright (non-blocking by default)

```yaml
  e2e:
    runs-on: ubuntu-latest
    # Remove `continue-on-error` once E2E is stable
    continue-on-error: true
    steps:
      - uses: actions/checkout@v4
      - uses: actions/setup-node@v4
        with:
          node-version: '20'
          cache: 'npm'
      - run: npm ci
      - name: Install Playwright browsers
        run: npx playwright install --with-deps chromium
      - name: Run E2E
        run: npx playwright test
        env:
          BASE_URL: ${{ secrets.STAGING_URL }}
```

---

## GitHub Secrets to Add

Tell the user:

> Add these in GitHub repo → Settings → Secrets and variables → Actions:
>
> | Secret | Value |
> |---|---|
> | `DATABASE_URL_TEST` | A separate test database URL — never point CI at prod or staging |
> | `RENDER_DEPLOY_HOOK_URL` | From Render service settings (only if using Render, not Vercel) |
> | Any env vars your tests need | Match whatever your app reads from `process.env` during tests |
>
> Note: Vercel users don't need any deploy secrets in GitHub — Vercel's GitHub integration handles deploys automatically.

---

## Rules

- **Always use `npm ci`** (not `npm install`) in CI — deterministic, respects lockfile.
- **Separate test DB** — never point CI at a production or staging database. Use `DATABASE_URL_TEST`.
- **Cache `node_modules`** via `actions/setup-node` `cache:` key — saves 30–60 seconds per run.
- **Don't put secrets in workflow YAML** — always `${{ secrets.NAME }}`.
- **E2E tests are non-blocking until stable** — flaky E2E blocking merges destroys team velocity.
- **Typecheck is separate from tests** — `tsc --noEmit` catches type errors that test runners often miss.

## If Something Goes Wrong

- **GitHub Actions workflow fails on first run** — check the runner logs for the exact failing step; missing env vars and wrong Node versions are the most common causes.
- **Deploy step fails with permission denied** — confirm deploy API keys are added to GitHub Secrets (not just local `.env`) and that secret names exactly match the workflow file.
- **Tests pass locally but fail in CI** — check for filesystem path case sensitivity, missing `NODE_ENV=test`, or tests that depend on local env vars not set in CI.