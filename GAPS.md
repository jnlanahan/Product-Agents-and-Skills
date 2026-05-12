# Gaps in Coverage

This file is the **single backlog of missing skills, missing agents, and missing modes** surfaced while designing the [eight workflows](WORKFLOWS.md). Each entry says **what's missing**, **why it matters**, **which workflows are affected**, and **priority**.

This is intentional inventory — we know what's not covered, so contributors and future-you can act on it.

For the full PDLC stages, see [PDLC_Phases.md](PDLC_Phases.md). For what *is* covered, see [AGENTS.md](AGENTS.md).

---

## Priority key

- **High** — actively biting users of multiple workflows
- **Medium** — useful, surfaces in one or two workflows, has a workaround
- **Low** — would be nice; the workaround (manual or scattered into other skills) is acceptable

---

## Discover phase

The PDLC explicitly flags Discover as **human-led** — this gap is partially intentional. But guided synthesis would still help.

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/market-scan`** — structured competitor analysis | Sometimes upstream of `/prd` | W2 | Low |

---

## Build phase

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/extract-module`** — pull a module out into a deep, testable shape | `/refactor` plans this but doesn't ship a recipe for it. | W5 | Low |

---

## Validate phase

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/load-test`** — performance / load testing with k6 or Artillery | Production-readiness without load testing is partial; refactors that target performance benefit from before/after benchmarks. | W2, W5, W7 | Low |
| **`/security-pentest`** — beyond static audit: simulated attack patterns | The current audit is static. Some hardening genuinely needs dynamic checks. | W7 | Low |

---

## Deploy phase

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/canary`** — progressive rollout configuration (percentage-based, region-based, cohort-based) | Manual today. Important for any `/deploy` where blast radius matters. `/feature-flag` covers PostHog-based percentage rollouts, but infrastructure-level canary (Render traffic splitting, Fly.io rolling deploys) is a separate concern. | W2, W3, W7 | Low |

---

## Learn phase

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/ab-test`** — A/B test setup + readout (beyond feature flags) | `/feature-flag` covers flag-based rollouts. A full A/B test skill would add statistical significance testing and readout from PostHog experiments. | W3 | Low |

---

## Cross-cutting

| Missing | Why useful | Priority |
|---|---|---|
| **Hooks for pipeline orchestration** — `SubagentStop` / `Stop` hooks chaining `/prd → /plan → /build-feature → /check-production → /deploy` automatically | Best practice #11 from the agent best-practices guide. Currently every transition is user-prompted. Best to build per-workflow chains rather than one global pipeline — that comes after the workflow library is stable. | Medium |
| **CLAUDE.md template** for projects that adopt this library | The current template lives in `/setup-project`; a standalone reference template would help retrofits and W4. | Medium |

---

## Principles for what we deliberately do **not** add

Some things look like gaps but are intentional:

1. **No `/migrate-between-auth-providers` skill.** Migrations between Firebase Auth ↔ Clerk ↔ NextAuth ↔ Supabase Auth are too risky to automate. Always manual. `/add-auth` extends what's already there; it never migrates.
2. **No `/migrate-between-payment-processors` skill.** Same reason. Stripe ↔ Paddle ↔ Lemon Squeezy migrations involve customer data and live subscriptions; automation makes mistakes here unacceptable.
3. **No `/auto-fix` skill** that reads `/check-production` output and applies fixes unattended. Fixes need user judgment in this library — best practice #10 says start read-only, and the human-in-the-loop pattern earns trust before we expand write capability.
4. **No `/refactor --auto`** mode. Same reason: refactors need a human reading every commit.

---

## How to propose a new skill

If a gap above is biting you, the simplest path to close it:

1. Pick the smallest possible scope (one job — best practice #4).
2. Draft `skills/<phase-letter>-<area>-<name>/SKILL.md` with action-oriented frontmatter description (best practice #2).
3. List the agents the skill calls; if a new diagnostic is needed, draft it as a read-only agent first (best practice #10).
4. Add it to [AGENTS.md](AGENTS.md), update affected [workflow files](workflows/), remove its row from this `GAPS.md`.
5. Verify on a real project (best practice #9 — evaluate before optimizing).
