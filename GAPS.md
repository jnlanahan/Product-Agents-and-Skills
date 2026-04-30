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

`/discover` (with `pain-point-miner` + `competitive-scanner`) covers the open-source-research subset. Interviews, sales-call mining, and honest TAM math remain human-led.

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/interview-synthesis`** — turn raw interview transcripts / sales-call notes / support-ticket dumps into structured insights that feed `/discover` and `/prd` | The web-research half is covered; the first-party voice-of-customer half isn't. Interviews are still the gold standard. | W1, W2, W4 | Medium |
| **TAM / market sizing** | Deliberately not automated — honest sizing numbers can't be web-scraped, and a skill that fabricated them would be worse than none. Stays human-led. | All | *(intentional)* |

---

## Build phase

The biggest gaps. Production-grade product workflows ([W2](workflows/W2-production-saas.md) and [W4](workflows/W4-migrate-to-production.md)) bump into these constantly.

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/add-email`** — Resend-first transactional email (templates, DKIM/SPF setup, send-on-event wiring) | The locked stack preference in `_stack-preferences.md` is Resend, but no skill exists. Email features get scattered into `/build-feature` ad hoc. | W2, W3, W4, W7 | **High** |
| **`/add-ai`** — Anthropic SDK + Files API + prompt caching + RAG patterns | AI integration is currently scattered into `/setup-project` Wave 7 and `/build-feature`. Deserves its own skill given how often it's needed. | W2, W3, W4, W8 | **High** |
| **`/setup-ci`** — GitHub Actions / Render auto-deploy / preview environments | `/check-production` *audits* CI but no skill *creates* it. | W2, W4 | Medium |
| **`/setup-tests`** — first test framework + first tests (Jest / Vitest / Playwright scaffolding) | Currently merged into `/build-feature`; standalone skill would help retrofits and `/migrate-from-vibe` outputs. | W3, W4, W5, W8 | Medium |
| **`/setup-project --personal`** — a personal-profile branch in `/setup-project` | [W8](workflows/W8-personal-tool.md) needs lighter setup (no Stripe, optional auth, SQLite). Currently requires telling the skill what to skip mid-conversation. | W8 | Medium |
| **`/extract-module`** — pull a module out into a deep, testable shape | `/refactor` plans this but doesn't ship a recipe for it. | W5 | Low |

---

## Validate phase

`/check-production` covers a lot but isn't the whole picture.

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/uat`** — structured user acceptance testing flow | Particularly important for [W4](workflows/W4-migrate-to-production.md), where the migration must preserve behavior the early users already rely on, and [W3](workflows/W3-add-feature.md) staged rollouts. | W2, W3, W4 | Medium |
| **`/accessibility`** — a11y audit (axe-core scan, keyboard navigation, screen-reader smoke test) | Not currently in `/check-production`'s 9 areas. | W2, W3, W7 | Medium |
| **`/load-test`** — performance / load testing with k6 or Artillery | Production-readiness without load testing is partial; refactors that target performance benefit from before/after benchmarks. | W2, W5, W7 | Low |
| **`/check-production --lite`** — a lite mode of the existing audit | `/check-production` is heavyweight; a 30-second sanity-check mode would unblock [W6](workflows/W6-fix-production-bug.md) and [W8](workflows/W8-personal-tool.md). | W6, W8 | Medium |
| **`/security-pentest`** — beyond static audit: simulated attack patterns | The current audit is static. Some hardening genuinely needs dynamic checks. | W7 | Low |

---

## Deploy phase

`/deploy` covers rollout. Rollback, canary, and on-call handoff are gaps.

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/rollback`** — codified rollback playbook (DB migrations, env changes, traffic cutover) | `/deploy` produces a rollout plan but the rollback path is currently manual. Most acute in [W6](workflows/W6-fix-production-bug.md) when a hotfix itself is the rollback condition. | W2, W3, W4, W6 | Medium |
| **`/canary`** — progressive rollout configuration (percentage-based, region-based, cohort-based) | Manual today. Important for any `/deploy` where blast radius matters. | W2, W3, W7 | Low |
| **`/runbook`** — generates an operational runbook from the just-deployed state | Useful for on-call handoff; particularly needed in [W4](workflows/W4-migrate-to-production.md) (first real deploy) and [W7](workflows/W7-audit-harden.md) (handover ready). | W2, W4, W7 | Medium |
| **`/feature-flag`** — wires feature-flag tooling and enforces flag-driven rollout in code | [W3](workflows/W3-add-feature.md) and [W7](workflows/W7-audit-harden.md) frequently want this. | W3, W7 | Medium |

---

## Learn phase

The single largest gap. Phase 7 currently has **zero skills**.

| Missing | Why useful | Workflows affected | Priority |
|---|---|---|---|
| **`/post-launch-review`** — metric review + retro a few weeks after deploy | Closes the loop on every workflow. Without it, learning is informal and incompletely captured. | All except W6 | **High** for ongoing improvement |
| **`/ab-test`** — A/B test setup + readout | Particularly relevant for [W3](workflows/W3-add-feature.md) feature decisions. | W3 | Low |
| **`/postmortem`** — structured postmortem after a severe bug or outage | Today done in a doc by hand; structured generation would be helpful. | W6 | Medium |

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
