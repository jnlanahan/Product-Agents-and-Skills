# Agents & Skills Index

This repository is a curated library of **Claude Code agents and skills** that drive a full Product Development Lifecycle (PDLC). Agents are read-only diagnostic helpers; skills are conversational workflows that orchestrate the work. Both are designed to be installed under `~/.claude/agents/` and `~/.claude/skills/` and used globally across projects.

For the lifecycle this library is built around, see [PDLC_Phases.md](PDLC_Phases.md). For composed end-to-end paths through these tools, see [WORKFLOWS.md](WORKFLOWS.md). For a list of known coverage gaps, see [GAPS.md](GAPS.md).

---

## Table of Contents

- [How to read this index](#how-to-read-this-index)
- [Agents (read-only diagnostics)](#agents-read-only-diagnostics)
- [Skills by PDLC phase](#skills-by-pdlc-phase)
  - [Stage 0 — Setup & Navigation](#stage-0--setup--navigation)
  - [Stage 1 — Discover](#stage-1--discover)
  - [Stage 2 — Define](#stage-2--define)
  - [Stage 3 — Design](#stage-3--design)
  - [Stage 4 — Build](#stage-4--build)
  - [Stage 5 — Validate](#stage-5--validate)
  - [Stage 6 — Deploy](#stage-6--deploy)
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

All eight agents are **read-only by design** (best practice #10). They detect, classify, and report. They never write. Skills call them to gather context before proposing changes.

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
- **Notes** — network-touching (WebFetch to the npm registry); narrows scope to production-relevant libraries to avoid noise

### `pain-point-miner`

- **Use when** — `/discover` calls this to surface real user complaints from public discussion (Reddit, HN, Stack Overflow, Indie Hackers, niche forums) about a problem topic
- **Tools** — `Read, Grep, Glob, WebSearch, WebFetch`
- **Output** — `PAIN POINT FINDINGS` with verbatim quotes, source URLs, frequency signal, severity reads, null-result notes, limitations
- **File** — [agents/pain-point-miner.md](agents/pain-point-miner.md)
- **Notes** — network-touching; capped at ~25 fetches per run; never fabricates quotes or counts

### `competitive-scanner`

- **Use when** — `/discover` calls this to map direct, indirect, and adjacent competitors for a product idea
- **Tools** — `Read, Grep, Glob, WebSearch, WebFetch`
- **Output** — `COMPETITIVE LANDSCAPE` with positioning, pricing, key features, gap candidates, vaporware/waitlist callouts
- **File** — [agents/competitive-scanner.md](agents/competitive-scanner.md)
- **Notes** — network-touching; capped at ~30 fetches per run; refuses to recommend a wedge — surfaces gaps only

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
- **Calls agents** — `stack-detector` (sanity check that this is greenfield)
- **File** — [skills/0-Setup-Project/SKILL.md](skills/0-Setup-Project/SKILL.md)

---

### Stage 1 — Discover

#### `/discover` — `1-Discover-Research`

- **Use when** — the user has an idea but hasn't validated whether the problem is real, who hurts, or what already exists. Run before `/prd` on a new idea, or before `/migrate-from-vibe` when the prototype's traction hasn't been re-validated.
- **Output** — `.claude/discover.md` with problem statement, target user, hypothesis, evidence-backed pain summary, competitive landscape, gap analysis, and a decision-shaped recommendation (proceed / sharpen / kill). Raw agent output preserved as appendices.
- **Calls agents** — `pain-point-miner`, `competitive-scanner` (in parallel)
- **File** — [skills/1-Discover-Research/SKILL.md](skills/1-Discover-Research/SKILL.md)

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

#### `/check-production` — `5-Validate-Production-Readiness`

- **Use when** — before a production launch, or after a big change to the critical path
- **Output** — severity-graded production-readiness report (Critical / High / Medium / Low) with `file:line` citations and recommended fix order. Orchestrates parallel scans then delegates the deep audit to `prod-readiness-auditor`.
- **Calls agents** — `stack-detector`, `codebase-classifier`, `secret-scanner`, `dependency-currency-checker`, `prod-readiness-auditor`
- **File** — [skills/5-Validate-Production-Readiness/SKILL.md](skills/5-Validate-Production-Readiness/SKILL.md)

---

### Stage 6 — Deploy

#### `/deploy` — `6-Deploy`

- **Use when** — first-time production deploy, or onboarding an existing app's deploy story
- **Output** — pre-flight checks, account setup, env vars, custom domain + SSL, third-party reconfigurations (webhooks, allowed origins, email DNS), post-deploy smoke tests, runbook. Heavy on step-by-step browser guidance.
- **Calls agents** — `secret-scanner` (pre-deploy gate), `prod-readiness-auditor` (if not already run)
- **File** — [skills/6-Deploy/SKILL.md](skills/6-Deploy/SKILL.md)

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
10. **Start read-only.** Reviewer / auditor / research agents first. The eight agents in this repo are all read-only; write capability lives in skills where the user is in the loop.

Plus two repo-level rules:

- **Hooks for pipeline orchestration** — see `settings.snippet.json` for the optional pattern.
- **CLAUDE.md at the repo root** — when this library is dropped into a project, the project's own CLAUDE.md should reference [WORKFLOWS.md](WORKFLOWS.md) and [_adaptation-playbook.md](skills/_adaptation-playbook.md) so any agent opening cold gets oriented.
