# Agents & Skills Index

This repository is a curated library of **Claude Code agents and skills** that drive a full Product Development Lifecycle (PDLC). Agents are read-only diagnostic helpers; skills are conversational workflows that orchestrate the work. Both are designed to be installed under `~/.claude/agents/` and `~/.claude/skills/` and used globally across projects.

For the lifecycle this library is built around, see [PDLC_Phases.md](PDLC_Phases.md). For composed end-to-end paths through these tools, see [WORKFLOWS.md](WORKFLOWS.md). For a list of known coverage gaps, see [GAPS.md](GAPS.md).

---

## Table of Contents

- [How to read this index](#how-to-read-this-index)
- [Agents (read-only diagnostics)](#agents-read-only-diagnostics)
- [Skills by PDLC phase](#skills-by-pdlc-phase)
  - [Stage 0 — Setup & Navigation](#stage-0--setup--navigation)
  - [Stage 2 — Define](#stage-2--define)
  - [Stage 3 — Design](#stage-3--design)
  - [Stage 4 — Build](#stage-4--build)
  - [Stage 5 — Validate](#stage-5--validate)
  - [Stage 6 — Deploy](#stage-6--deploy)
  - [Stage 7 — Learn](#stage-7--learn)
- [Supporting reference files](#supporting-reference-files)
- [Design principles](#design-principles)

---

## How to read this index

Every entry follows the same shape:

- **Name** — the filename / slash-command users type
- **Use when** — the action-oriented delegation trigger from frontmatter
- **Tools** — the explicit tool allowlist (least-privilege)
- **Output** — the structured artifact this produces
- **Best-practice notes** — flags any deviation from the [10 best practices](#design-principles)

Source of truth for each entry is the `description` field in the file's YAML frontmatter — this index is hand-curated to match.

---

## Agents (read-only diagnostics)

All six agents are **read-only by design** (best practice #10). They detect, classify, and report. They never write. Skills call them to gather context before proposing changes.

### `stack-detector`

- **Use when** — at the start of any skill that needs to know what stack the current project uses
- **Tools** — `Read, Grep, Glob`
- **Output** — structured `STACK PROFILE` block: framework, DB/ORM, auth, payments, AI, monitoring, deploy target
- **File** — [agents/stack-detector.md](agents/stack-detector.md)
- **Notes** — cheap and fast; the entry-point read for almost every skill

### `codebase-classifier`

- **Use when** — after `stack-detector`, whenever a skill's behavior depends on whether the codebase is **greenfield**, **wired**, or **vibe-coded**
- **Tools** — `Read, Grep, Glob`
- **Output** — one-word verdict (`greenfield` / `wired` / `vibe-coded`) + confidence + adaptation hint
- **File** — [agents/codebase-classifier.md](agents/codebase-classifier.md)
- **Notes** — drives whether a skill installs fresh, extends existing, or treats the codebase delicately per `_adaptation-playbook.md`

### `pattern-finder`

- **Use when** — before a skill writes a new file (route, component, hook, storage method, test) — find the closest existing example so the new file matches local style
- **Tools** — `Read, Grep, Glob`
- **Output** — `PATTERN` block: location, naming, imports, validation, error handling, response shape, auth wiring
- **File** — [agents/pattern-finder.md](agents/pattern-finder.md)
- **Notes** — critical for vibe-coded apps where convention is implicit and inconsistent

### `prod-readiness-auditor`

- **Use when** — `/check-production` (the validate skill) calls this for the deep 9-area audit — secrets, auth, payments, files, DB, API, errors, AI, CI/CD
- **Tools** — `Read, Grep, Glob`
- **Output** — severity-graded report (Critical / High / Medium / Low) with `file:line` citations, impact, and fix suggestion per finding
- **File** — [agents/prod-readiness-auditor.md](agents/prod-readiness-auditor.md)
- **Notes** — heavy; not for casual checks. `/check-production` is the only orchestrator that should invoke it.

### `secret-scanner`

- **Use when** — before any production deploy and during `/check-production`
- **Tools** — `Read, Grep, Glob, Bash`
- **Output** — `SECRET SCAN REPORT` with truncated evidence and rotation actions for each finding
- **File** — [agents/secret-scanner.md](agents/secret-scanner.md)
- **Notes** — scans both working tree AND git history; reports only — never deletes or rotates

### `dependency-currency-checker`

- **Use when** — `/next-steps` and `/check-production` use this to flag stack-relevant dependencies that have drifted from their current major versions
- **Tools** — `Read, Grep, Glob, Bash, WebFetch`
- **Output** — `CURRENCY REPORT` with risk and effort estimates per dependency
- **File** — [agents/dependency-currency-checker.md](agents/dependency-currency-checker.md)
- **Notes** — only one with network access (WebFetch to the npm registry); narrows scope to production-relevant libraries to avoid noise

### `model-selector`

- **Use when** — `/add-ai` calls this before choosing a Claude model. Reads codebase context, asks up to 5 targeted questions about the AI use case, and recommends the right tier.
- **Tools** — `Read, Grep, Glob`
- **Output** — `AI REQUIREMENTS PROFILE` block: app domain, primary task, latency/reasoning/context/volume classification, recommended model (Haiku/Sonnet/Opus), streaming preference, prompt caching flag, LangSmith eval recommendation
- **File** — [agents/model-selector.md](agents/model-selector.md)
- **Notes** — key decision rule: image-gen orchestrator or real-time/high-volume → Haiku; most features → Sonnet (default); deep reasoning → Opus

### `accessibility-auditor`

- **Use when** — `/accessibility` calls this to scan all component and template files for WCAG 2.1 AA violations
- **Tools** — `Read, Grep, Glob`
- **Output** — `ACCESSIBILITY REPORT` with severity-graded findings (Critical/High/Medium/Low) and `file:line` citations per finding
- **File** — [agents/accessibility-auditor.md](agents/accessibility-auditor.md)
- **Notes** — static analysis only (~30% of a11y issues); the skill always appends a manual testing checklist for the remaining 70%

---

## Skills by PDLC phase

Skill folders are prefixed with the PDLC stage number so they sort by lifecycle order.

### Stage 0 — Setup & Navigation

#### `/next-steps` — `0-Always-Next-Steps`

- **Use when** — anytime you want to see the production-readiness state, what changed since the last check, and which skills to run next. The default "what should I do next?" command.
- **Output** — updated `.claude/next-steps.md` with the project's living journey to production
- **Calls agents** — `stack-detector`, `codebase-classifier`, `dependency-currency-checker`
- **File** — [skills/0-Always-Next-Steps/SKILL.md](skills/0-Always-Next-Steps/SKILL.md)

#### `/setup-project` — `0-Setup-Project`

- **Use when** — starting a brand-new SaaS project from empty or fresh-scaffold state. **Not** for existing projects.
- **Output** — scaffolded project in disciplined waves (one commit per wave), `CLAUDE.md` with skills index, third-party accounts wired, git/GitHub conventions seeded
- **Flags** — `--personal`: lighter stack (SQLite, optional auth, no Stripe/Storage) for personal tools and internal dashboards
- **Calls agents** — `stack-detector` (sanity check that this is greenfield)
- **File** — [skills/0-Setup-Project/SKILL.md](skills/0-Setup-Project/SKILL.md)

---

### Stage 2 — Define

#### `/prd` — `2-Define-PRD`

- **Use when** — the user wants to create a PRD synthesized from the current conversation and codebase understanding (does NOT interview)
- **Output** — `.claude/prd.md` with problem, solution, personas, user stories, functional/non-functional requirements, success metrics, risks, rollout plan, out-of-scope
- **File** — [skills/2-Define-PRD/SKILL.md](skills/2-Define-PRD/SKILL.md)

#### `/plan` — `2-Define-Plan`

- **Use when** — turning a PRD (or current conversation context) into an executable implementation plan
- **Output** — `.claude/plan.md`: vertical slices (tracer bullets), TDD strategy per slice, commit sequencing by layer (schema → storage → routes → hooks → components). `/build-feature` reads this.
- **File** — [skills/2-Define-Plan/SKILL.md](skills/2-Define-Plan/SKILL.md)
- **Supporting docs** — `deep-modules.md`, `interface-design.md`, `mocking.md`, `tests.md`, `refactoring.md`

#### `/refactor` — `2-Define-Refactor`

- **Use when** — refactoring code, in either **find** mode ("where's our shallow code?") or **plan** mode (a refactor is decided; safely sequence small commits)
- **Output** — `.claude/refactor-plan.md` with tiny commits + Claude-Code refactoring best practices
- **File** — [skills/2-Define-Refactor/SKILL.md](skills/2-Define-Refactor/SKILL.md)
- **Supporting docs** — `LANGUAGE.md`, `INTERFACE-DESIGN.md`, `DEEPENING.md`

#### `/glossary` — `2-Define-Glossary`

- **Use when** — extracting and formalizing project domain terms; flagging ambiguous terms; proposing canonical names
- **Output** — `.claude/glossary.md` with term tables, relationships, example dialogue
- **File** — [skills/2-Define-Glossary/SKILL.md](skills/2-Define-Glossary/SKILL.md)

#### `/grill-me` — `2-Define-Grill-Me`

- **Use when** — stress-testing a plan or design with relentless one-question-at-a-time interrogation until shared understanding is reached
- **Output** — surfaced unknowns, decided trade-offs, sharper plan
- **File** — [skills/2-Define-Grill-Me/SKILL.md](skills/2-Define-Grill-Me/SKILL.md)

---

### Stage 3 — Design

#### `/prototype` — `3-Design-Prototype`

- **Use when** — designing or visualizing a feature's UI before building it
- **Output** — three radically different clickable HTML prototypes at `prototypes/variant-A.html`, `variant-B.html`, `variant-C.html` (TailwindCSS via CDN, mock data, fake auth/API). User picks one; the chosen variant becomes the design reference for `/build-feature`.
- **File** — [skills/3-Design-Prototype/SKILL.md](skills/3-Design-Prototype/SKILL.md)

---

### Stage 4 — Build

#### `/code-map` — `4-Build-Code-Map`

- **Use when** — unfamiliar with a section of code, or you want a higher-level map of how an area fits the bigger picture
- **Output** — module map, public interfaces surfaced (internals hidden), data-flow narrative
- **File** — [skills/4-Build-Code-Map/SKILL.md](skills/4-Build-Code-Map/SKILL.md)

#### `/setup-database` — `4-Build-Database`

- **Use when** — first-time DB setup, or adding a table / column / index / migration
- **Output** — generated migration → review → apply → verify, with explicit warnings on destructive operations. Detects ORM (Drizzle / Prisma / Kysely / raw SQL) and adapts.
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-Database/SKILL.md](skills/4-Build-Database/SKILL.md)

#### `/add-auth` — `4-Build-Auth`

- **Use when** — adding or extending authentication (sign-up, sign-in, social login, MFA, organizations, RBAC)
- **Output** — wired auth — Firebase Auth preferred for greenfield; adapts to Clerk / NextAuth / Supabase Auth / custom JWT if detected (never migrates)
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-Auth/SKILL.md](skills/4-Build-Auth/SKILL.md)

#### `/add-payment` — `4-Build-Payments`

- **Use when** — adding or extending payments (subscriptions, billing, customer portal)
- **Output** — Stripe wired (or extends existing Stripe / surfaces a different processor); always includes webhook signature verification, idempotency, Customer Portal
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-Payments/SKILL.md](skills/4-Build-Payments/SKILL.md)

#### `/add-files` — `4-Build-File-Storage`

- **Use when** — adding or extending file uploads, file sharing, folders, tags, image transforms, per-user quotas
- **Output** — Firebase Storage wired (or extends existing S3 / R2 / UploadThing); enforces server-side validation, ownership checks, magic-byte MIME verification
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-File-Storage/SKILL.md](skills/4-Build-File-Storage/SKILL.md)

#### `/add-monitoring` — `4-Build-Monitoring`

- **Use when** — wiring observability before any production launch
- **Output** — both Sentry (errors, stack traces, performance, source maps) AND PostHog (analytics, replay, flags, funnels). Walks through account setup, env vars, verification with real test events. Identifies authenticated users in both.
- **File** — [skills/4-Build-Monitoring/SKILL.md](skills/4-Build-Monitoring/SKILL.md)

#### `/add-ai` — `4-Build-AI`

- **Use when** — adding AI / LLM capabilities to any project
- **Output** — Anthropic SDK wired, correct Claude model selected (via `model-selector`), prompt caching enabled, LangSmith tracing + eval scaffold added
- **Model selection logic** — image-gen orchestrator or real-time/simple → `claude-haiku-4-5`; most features → `claude-sonnet-4-6`; deep reasoning → `claude-opus-4-7`
- **Calls agents** — `stack-detector`, `model-selector`
- **File** — [skills/4-Build-AI/SKILL.md](skills/4-Build-AI/SKILL.md)

#### `/add-email` — `4-Build-Email`

- **Use when** — adding transactional email (welcome, password reset, notifications) to a project
- **Output** — Resend wired (or extends detected provider), React Email templates, DNS setup walkthrough, delivery verified
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-Email/SKILL.md](skills/4-Build-Email/SKILL.md)

#### `/setup-ci` — `4-Build-CI`

- **Use when** — a project has no CI and needs GitHub Actions for typecheck, tests, lint, and/or auto-deploy
- **Output** — `.github/workflows/ci.yml` (and optionally `deploy.yml`), GitHub Secrets list, verified passing run
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-CI/SKILL.md](skills/4-Build-CI/SKILL.md)

#### `/setup-tests` — `4-Build-Tests`

- **Use when** — a project has no test framework; scaffolds Vitest (Next.js/Vite) or Jest (Express/Node) and writes the first meaningful tests
- **Output** — test framework config, setup files, `npm test` passing with ≥3 real tests, coverage script
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-Tests/SKILL.md](skills/4-Build-Tests/SKILL.md)

#### `/build-feature` — `4-Build-Feature`

- **Use when** — implementing a new feature in coherent TDD layers
- **Output** — schema → storage → routes → hooks → components, one commit per layer with tests at each. Reads `.claude/plan.md` if present; otherwise interviews briefly.
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/4-Build-Feature/SKILL.md](skills/4-Build-Feature/SKILL.md)

#### `/migrate-from-vibe` — `4-Build-Migrate-From-Vibe`

- **Use when** — moving a project off a vibe-coding platform (Replit, V0, Lovable, Bolt, Cursor-only, ChatGPT-generated) onto a real local stack
- **Output** — extracted working app, ported in waves to the user's preferred stack, env vars and integrations remapped. Inconsistencies flagged as out-of-scope rather than fixed inline.
- **Calls agents** — `stack-detector`, `codebase-classifier`, `pattern-finder`
- **File** — [skills/4-Build-Migrate-From-Vibe/SKILL.md](skills/4-Build-Migrate-From-Vibe/SKILL.md)

---

### Stage 5 — Validate

#### `/triage` — `5-Validate-Triage`

- **Use when** — the user reports a bug or wants to investigate a problem
- **Output** — root-cause hypothesis with `file:line` evidence, 2+ proposed fixes (root cause + workaround), full bug report at `.claude/bugs/<short-name>.md`. Replaces older bug-investigation, bug-report, and QA-session skills.
- **File** — [skills/5-Validate-Triage/SKILL.md](skills/5-Validate-Triage/SKILL.md)

#### `/uat` — `5-Validate-UAT`

- **Use when** — before migrating users to a new version or shipping a feature that changes existing behavior; generates a UAT checklist and walks through each scenario
- **Output** — UAT report at `.claude/uat-<feature>-<date>.md` with pass/fail/blocked per scenario and a PASS / FAIL / CONDITIONAL decision
- **Calls agents** — `stack-detector`
- **File** — [skills/5-Validate-UAT/SKILL.md](skills/5-Validate-UAT/SKILL.md)

#### `/accessibility` — `5-Validate-Accessibility`

- **Use when** — auditing a project for WCAG 2.1 AA compliance
- **Output** — prioritized a11y fix list with effort estimates; optional in-place fixes (one per commit); mandatory manual testing checklist
- **Calls agents** — `stack-detector`, `accessibility-auditor`
- **File** — [skills/5-Validate-Accessibility/SKILL.md](skills/5-Validate-Accessibility/SKILL.md)

#### `/check-production` — `5-Validate-Production-Readiness`

- **Use when** — before a production launch, or after a big change to the critical path
- **Output** — severity-graded production-readiness report (Critical / High / Medium / Low) with `file:line` citations and recommended fix order. Orchestrates parallel scans then delegates the deep audit to `prod-readiness-auditor`.
- **Flags** — `--lite`: fast 30-second sanity check (secret-scanner + build + health check only); for hotfixes and personal tools
- **Calls agents** — `stack-detector`, `codebase-classifier`, `secret-scanner`, `dependency-currency-checker`, `prod-readiness-auditor`
- **File** — [skills/5-Validate-Production-Readiness/SKILL.md](skills/5-Validate-Production-Readiness/SKILL.md)

---

### Stage 6 — Deploy

#### `/feature-flag` — `6-Deploy-Feature-Flag`

- **Use when** — gating a new feature for staged rollout, A/B testing, or kill-switch control
- **Output** — PostHog feature flag wired (client + server), flag-guarded code, staged rollout plan at `.claude/flag-<name>-rollout.md`
- **Calls agents** — `stack-detector`, `pattern-finder`
- **File** — [skills/6-Deploy-Feature-Flag/SKILL.md](skills/6-Deploy-Feature-Flag/SKILL.md)

#### `/rollback` — `6-Deploy-Rollback`

- **Use when** — a production deploy has introduced a regression, or proactively before a risky deploy to document the rollback path
- **Output** — numbered rollback runbook at `.claude/rollback-<date>.md` covering code, DB migrations, env vars, and traffic cutover; walks through execution on request
- **File** — [skills/6-Deploy-Rollback/SKILL.md](skills/6-Deploy-Rollback/SKILL.md)

#### `/runbook` — `6-Deploy-Runbook`

- **Use when** — after a successful production deploy to generate an operational runbook for on-call handoff
- **Output** — `RUNBOOK.md` at project root covering health check, startup/shutdown, env vars, common failure modes + fixes, alert response, rollback steps
- **Calls agents** — `stack-detector`
- **File** — [skills/6-Deploy-Runbook/SKILL.md](skills/6-Deploy-Runbook/SKILL.md)

#### `/deploy` — `6-Deploy`

- **Use when** — first-time production deploy, or onboarding an existing app's deploy story
- **Output** — pre-flight checks, account setup, env vars, custom domain + SSL, third-party reconfigurations (webhooks, allowed origins, email DNS), post-deploy smoke tests, runbook. Heavy on step-by-step browser guidance.
- **Calls agents** — `secret-scanner` (pre-deploy gate), `prod-readiness-auditor` (if not already run)
- **File** — [skills/6-Deploy/SKILL.md](skills/6-Deploy/SKILL.md)

---

### Stage 7 — Learn

#### `/post-launch-review` — `7-Learn-Post-Launch-Review`

- **Use when** — 2–4 weeks after a production launch or 1–2 weeks after a major feature ship to close the learn loop
- **Output** — structured review at `.claude/post-launch-review-<date>.md`: results vs. goals table, Start/Stop/Continue retro, ranked action items, next-iteration `/prd` candidates
- **File** — [skills/7-Learn-Post-Launch-Review/SKILL.md](skills/7-Learn-Post-Launch-Review/SKILL.md)

#### `/postmortem` — `7-Learn-Postmortem`

- **Use when** — after a production outage, severe bug, data incident, or security exposure
- **Output** — blameless postmortem at `.claude/postmortem-<date>-<slug>.md`: incident timeline, 5 Whys root cause analysis, contributing factors, action items with owners and due dates
- **File** — [skills/7-Learn-Postmortem/SKILL.md](skills/7-Learn-Postmortem/SKILL.md)

---

## Supporting reference files

These aren't skills themselves; they're shared knowledge that skills reference.

| File | Purpose |
| --- | --- |
| [PDLC_Phases.md](PDLC_Phases.md) | Defines all 7 PDLC phases (Discover, Define, Design, Build, Validate, Deploy, Learn) — activities, outputs, exit criteria |
| [skills/_stack-preferences.md](skills/_stack-preferences.md) | The user's locked greenfield stack: Firebase Auth, Stripe, Neon + Drizzle, Firebase Storage, PostHog + Sentry, Resend, etc. Read by `/setup-project` and `/add-*` skills when nothing is detected. |
| [skills/_adaptation-playbook.md](skills/_adaptation-playbook.md) | Five core rules every skill follows when deciding whether to install fresh, extend existing, or stay out of the way. Critical for handling vibe-coded apps without breaking them. |
| [settings.snippet.json](settings.snippet.json) | Optional Claude Code hook config to merge into `~/.claude/settings.json`. Provides a session-start reminder about `stack-detector` and runs `npm run check` after edits. |

---

## Design principles

Every agent and skill in this repo is checked against these ten best practices.

1. **Pick the right mechanism for the job.** Subagents isolate context for one-off delegated work. Skills package reusable procedural knowledge. Hooks orchestrate pipelines. MCP servers connect external systems. — *In this repo: read-only diagnostic work is always a subagent; recurring user-facing workflows are always a skill.*
2. **Write the description as a delegation trigger, not a summary.** Action-oriented "MUST BE USED for X" / "Use after Y to produce Z."
3. **Scope tools explicitly.** Every agent declares `tools:` with the minimum needed. Reviewers get `Read, Grep, Glob`; implementers get write tools; only `dependency-currency-checker` touches the network.
4. **One subagent, one job.** If the goal, input, output, and handoff don't fit in a sentence, the scope is wrong.
5. **Stateless invocations.** No memory between runs. Every needed file path, error, or decision is passed in the invocation prompt.
6. **Behavioral rules in the body, not the description.** Concrete constraints — line limits, error patterns, output structure, prohibited actions — live in the system prompt body.
7. **Demand structured output.** Every agent returns a labeled block (`STACK PROFILE`, `PATTERN`, `SECRET SCAN REPORT`, etc.) so callers parse cleanly.
8. **Skills for procedural knowledge that repeats.** A SKILL.md plus optional scripts and references — not prompts re-explained each session.
9. **Evaluate before optimizing.** Run on real tasks, find where it fails, fix what actually breaks.
10. **Start read-only.** Reviewer / auditor / research agents first. The six agents in this repo are all read-only; write capability lives in skills where the user is in the loop.
11. **Creator-verifier separation.** The agent (or skill) that builds a feature should not be the same one that judges whether it's complete. `/build-feature` ends with an explicit handoff to the user + `/check-production` (a separate agent with no build context) for verification. Never let the implementer self-certify.

Plus two repo-level rules:

- **Hooks for pipeline orchestration** — see `settings.snippet.json` for the optional pattern.
- **CLAUDE.md at the repo root** — when this library is dropped into a project, the project's own CLAUDE.md should reference [WORKFLOWS.md](WORKFLOWS.md) and [_adaptation-playbook.md](skills/_adaptation-playbook.md) so any agent opening cold gets oriented.
