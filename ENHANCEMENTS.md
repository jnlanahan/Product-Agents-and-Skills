# Enhancement Plan

A specification for upgrading this PM workflow library without compromising its lightweight, detection-driven character. This document is the source of truth for implementation.

---

## Context

This codebase is a PM workflow library built from Claude Code skills and agents. Its working principles, observed from the existing code:

- **Single-file skills** — each skill is one `SKILL.md`, low cognitive overhead.
- **Detection-then-adapt** — diagnostic agents (`stack-detector`, `codebase-classifier`, `pattern-finder`) read the project state before any skill writes code.
- **Creator-verifier separation** — the agent that builds does not certify; an independent agent runs validation with fresh context.
- **`.claude/` artifact convention** — all PM artifacts live in `.claude/` (`prd.md`, `plan.md`, etc.).
- **Vertical-slice TDD** — implementation moves one slice at a time, with `LAYER HANDOFF` blocks between layers.

Real gaps to address: no discovery scaffolding, no architecture phase, no telemetry skill, no project-level state awareness, no session-continuity beyond file existence, and weak entry-points for PMs who don't already know what to invoke.

---

## Design principles (do not violate)

1. **Stay single-file** — every new skill is one `SKILL.md`. Exceptions in Phase 2 and Phase 3 are explicitly marked.
2. **Stay additive** — never hide or remove skills based on detected state. Surface warnings, never gate.
3. **Stay detection-driven** — declared workflows are forbidden. Project mode is read from `.claude/` state, never selected up front.
4. **Stay reversible** — any state file is editable markdown. No databases, no MCP servers, no daemons.
5. **Lean on Claude Code primitives** — `CLAUDE.md`, auto memory, AutoDream. Don't reinvent what ships with the tool.

## What NOT to do

- Do not introduce hierarchical config systems with external resolvers (e.g., TOML files resolved by Python scripts at skill invocation). The lightweight `.claude/context.md` pattern below replaces this need.
- Do not introduce interactive persona-shell agents that present menus and route to skills. The existing diagnostic agent pattern (read-only, silent, called by skills) is the correct one and must not be diluted.
- Do not convert existing single-file skills to step-file directories wholesale. Selective decomposition only for the 2 explicitly listed skills in Phase 2 and Phase 3.
- Do not introduce a `/select-workflow` skill at project start. Detection replaces declaration.
- Do not hide skills based on mode. The mode-detector adds signal, never subtracts capability.
- Do not build a vector DB or memory MCP. Markdown files in `.claude/` are sufficient.

---

# Phase 1 — Memory and Project Intelligence Foundation

Everything else depends on this phase. Build it first.

## 1.1 — Update `CLAUDE.md` at project root

Path: `CLAUDE.md` (project root, not inside `.claude/`).

Create or update with the following sections.

**Memory protocol section:**

```markdown
## Memory protocol

Every skill in this project follows this discipline:

1. Read `.claude/progress.md` before beginning work. It is the project's
   narrative log — what was done, what's next, what's blocked.
2. Read any artifacts listed as inputs (`.claude/prd.md`, `.claude/plan.md`,
   `.claude/architecture.md`, `.claude/context.md`, etc.) before generating output.
3. On completion, append a 3–5 line entry to `.claude/progress.md`:
   timestamp, skill name, output path, key decisions, suggested next step.

Assume the context window resets at any moment. Anything not written to
`.claude/` is lost. Skills must not rely on conversational memory to carry
state forward.

If `.claude/progress.md` does not exist, create it with a header before
appending the first entry.
```

**Project context pointer:**

```markdown
## Project context

Persistent project context lives at `.claude/context.md`. Read it at session
start. It contains stack choices, conventions, constraints, and stakeholder
notes that don't change frequently.
```

## 1.2 — `.claude/progress.md` (narrative log)

Path: `.claude/progress.md`. Created by `/start` (1.5) or by any skill that finds it missing.

Format: append-only markdown. Each entry:

```markdown
## 2026-05-11 14:23 — /plan completed
- Output: `.claude/plan.md`
- Vertical slices: 4 (auth, data layer, UI, integration tests)
- Decisions: Postgres over SQLite per stack-detector recommendation
- Next likely: /build-feature for slice 1 (auth)
- Open: rate limiting strategy deferred
```

Never overwritten. Skills append; humans curate by editing directly when entries get stale.

## 1.3 — `.claude/context.md` (project context layer)

Path: `.claude/context.md`. Created by `/start`.

Contents (sections, all optional):
- `## Stack` — chosen technologies and versions
- `## Conventions` — naming, file layout, commit style
- `## Constraints` — compliance, performance, team
- `## Stakeholders` — who reviews, who approves
- `## Glossary pointer` — link to `.claude/glossary.md` if it exists

A single editable file with no resolver and no scope hierarchy. All persistent project context that doesn't change frequently lives here.

## 1.4 — New agent: `project-state-detector`

Path: `agents/project-state-detector.md`.

**Role:** Read-only diagnostic agent. Sits alongside `stack-detector`, `codebase-classifier`, `pattern-finder`.

**Inputs:** the `.claude/` directory contents, repo state, recent git activity.

**Output (structured):**
```
MODE: Discovery | Definition | Design | Build | Hardening | Launch | Operating
MATURITY: entering | deep | exiting
RECOMMENDED_NEXT: <skill name>
OFF_PATTERN_SKILLS: <list — skills the PM could still invoke but that are off-mode>
SIGNALS: <one-line evidence for the mode call>
```

**Mode-detection rules:**
- No `.claude/prd.md` or empty → Discovery
- `.claude/prd.md` exists, no `.claude/plan.md` → Definition
- `.claude/plan.md` exists, no implementation code linked → Design
- Implementation code linked, no validation artifact → Build
- Validation passing, no deploy artifact → Hardening
- Deploy artifact exists, no post-launch review → Launch
- Post-launch review exists → Operating

These rules are starting points. The detector should also use git activity and artifact freshness (timestamps) when file existence alone is ambiguous.

## 1.5 — New skill: `/start`

Path: `skills/0-Start/SKILL.md`. Single file.

**Purpose:** Initialize a new project. Runs once at project init. Additive only — never destructive.

**Behavior:**
1. Check for `.claude/` directory; create if missing.
2. Ask 3–5 questions:
   - What are you doing? (new product / new feature / rearchitecture / bug fix / exploration)
   - What do you already have? (just an idea / discovery notes / PRD / existing code / live product)
   - Team context? (solo / small team / cross-functional)
   - Primary deliverable expected?
   - Anything else essential? (regulated industry, design partner, etc.)
3. Write answers to `.claude/context.md` under appropriate sections.
4. Seed `.claude/progress.md` with a header and first entry.
5. Suggest a starting skill based on answers (without forcing).

**Constraint:** No workflow selection. Don't ask "which workflow do you want." Just capture context and suggest.

**Re-runnable:** If `.claude/context.md` already exists, offer to update specific sections rather than overwrite.

## 1.6 — New skill: `/resume`

Path: `skills/0-Resume/SKILL.md`. Single file.

**Purpose:** Recover session state at the start of a new chat. Run by the PM when opening a project that has prior history.

**Behavior:**
1. Read last 10 entries of `.claude/progress.md`.
2. Call `project-state-detector`.
3. Output a 5–10 line summary: current mode, last activity, open threads, recommended next step.
4. Do not modify any files.

This is the human-facing counterpart to the memory protocol. The protocol guarantees the log gets written; `/resume` makes sure the PM reads it.

## 1.7 — Update existing `0-Always-Next-Steps` → `/next`

Path: `skills/0-Next/SKILL.md` (rename from `0-Always-Next-Steps` if that exists).

**Purpose:** Live, state-aware dashboard. Replaces the previously static reference.

**Behavior:**
1. Call `project-state-detector`.
2. Read `.claude/progress.md` for last 5 entries.
3. Output:
   - Current mode and maturity
   - What artifacts exist in `.claude/`
   - What's stale (artifacts not updated in 14+ days while project is active)
   - Recommended next skill with one-line rationale
   - Optional skills available now
   - Off-pattern skills (still callable, flagged)
4. Do not modify any files.

Runnable anytime the PM is disoriented. Output should fit on one screen.

## 1.8 — Skill pre-flight / post-flight protocol

Modify every existing skill in `skills/` to add at the top:

```markdown
## Pre-flight
- Read `.claude/progress.md`
- Read `.claude/context.md` if it exists
- Read inputs listed in this skill's frontmatter
- Call `project-state-detector` and surface off-mode warning if applicable
  (do NOT block; warn and proceed if user confirms)

## Post-flight
- Append entry to `.claude/progress.md`:
  - Timestamp, skill name, output path
  - Key decisions made
  - Suggested next-step skill
```

The redundancy with `CLAUDE.md`'s memory protocol is intentional. Skills carry the protocol with them when read in isolation.

**Read each existing skill carefully before modifying.** Some skills may have idiosyncratic structures the protocol needs to slot into rather than replace. Do not destroy existing skill logic to make room for the protocol header.

---

# Phase 2 — UX Cliff Fixes

The hardest UX failure today: `/prd` assumes the PM knows the problem cold. If they don't, they're stuck. Fix this before doing anything else in this phase.

## 2.1 — Smart routing at the top of `/prd`

Path: `skills/2-Define-PRD/SKILL.md`.

Add a routing check at the top of the skill:

```markdown
## Pre-flight routing

Before running the PRD synthesis, evaluate whether the user has enough
context to produce a useful PRD:

- Is a specific customer or persona named?
- Is a concrete problem articulated?
- Is the scope of "what to build" clear?

If two or more of these are missing or vague, suggest `/discover` instead
of proceeding. Phrase it as a suggestion, not a block:

"Your inputs look thin for a PRD synthesis. I can either proceed with
what's here, or run /discover to surface the problem before structuring
it. Which do you prefer?"

Proceed with the PRD if the user confirms, regardless of input quality.
The routing is a suggestion, never a gate.
```

## 2.2 — New skill: `/discover` (step-decomposed)

Path: `skills/1-Discover/SKILL.md` plus `skills/1-Discover/steps/`.

**Selective step decomposition justified here** because discovery is typically a multi-session, exploratory workflow with natural pause points. This is one of only two skills in the entire library that should be step-decomposed.

**Structure:**
- `SKILL.md` — top-level orchestrator. Reads progress, decides which step to run next.
- `steps/step-01-frame.md` — frame the problem space
- `steps/step-02-customer.md` — articulate the specific customer / persona
- `steps/step-03-problem.md` — sharpen the problem statement
- `steps/step-04-stakes.md` — surface why it matters now
- `steps/step-05-bridge.md` — produce a problem statement ready for `/prd`

**State tracking:** `.claude/discovery-notes.md` with frontmatter:
```yaml
---
stepsCompleted: ['frame', 'customer']
lastStep: customer
nextStep: problem
---
```

**Output:** `.claude/discovery-notes.md` (a complete document by the end). `/prd` reads this if present and uses it as input to skip its own front-door interview.

**Optional, skippable.** PMs who arrive with problems confirmed never invoke this.

---

# Phase 3 — Architectural Gaps

## 3.1 — New skill: `/architect` (step-decomposed)

Path: `skills/3-Architect/SKILL.md` plus `skills/3-Architect/steps/`.

**Selective step decomposition justified here** because architecture decisions are gated and reviewable — each step ends at a checkpoint the PM should consciously pass. This is the second of two skills in the library that should be step-decomposed.

**Structure:**
- `SKILL.md` — orchestrator
- `steps/step-01-detect-stack.md` — calls `stack-detector`, summarizes current state
- `steps/step-02-data-model.md` — produces data model decisions
- `steps/step-03-integrations.md` — surfaces integration points and constraints
- `steps/step-04-tradeoffs.md` — captures key tradeoffs and rejected alternatives
- `steps/step-05-output.md` — assembles `.claude/architecture.md`

**Output:** `.claude/architecture.md`.

**Skippable** for trivial features. Called explicitly when novel architecture is at stake.

**Reads:** `.claude/prd.md`, `.claude/context.md`, output of `stack-detector` and `codebase-classifier`.

## 3.2 — New skill: `/measure`

Path: `skills/2-Define-Measurement/SKILL.md`. Single file.

**Purpose:** Produce a telemetry and measurement plan. Fills the gap documented in `PDLC_Phases.md` (measurement plan listed as Define-phase deliverable, no skill implements it).

**Output:** `.claude/measurement.md` with sections:
- Success metrics (what defines this feature working)
- Event schema (what gets logged, with field types)
- Telemetry plan (where events go, retention, dashboards)
- Failure signals (what triggers alerts)

**Pairs with:** existing `/add-monitoring` skill.

**Optional, skippable.** Data-driven PMs invoke it; others don't.

---

# Phase 4 — Polish

## 4.1 — New skill: `/skills` (skill discovery)

Path: `skills/0-Skills/SKILL.md`. Single file.

**Purpose:** Help the PM discover available skills without memorizing slash commands.

**Behavior:**
1. List all skills in `skills/` with one-line descriptions read from each `SKILL.md` frontmatter.
2. Call `project-state-detector` and highlight which skills make sense now.
3. Group by PDLC phase per `PDLC_Phases.md`.

**Does not modify any files.**

## 4.2 — New skill: `/handoff`

Path: `skills/6-Handoff/SKILL.md`. Single file.

**Purpose:** Package PRD + plan + validation contract into a standalone document for stakeholders without codebase access (contractors, outsourced dev, executives).

**Reads:** `.claude/prd.md`, `.claude/plan.md`, any `.claude/architecture.md`.

**Output:** `.claude/handoff-<feature-name>.md` — a single document that reads end-to-end without requiring source access. Includes acceptance criteria expressed as PM-readable validation contracts.

## 4.3 — New agent: `design-tokens-detector`

Path: `agents/design-tokens-detector.md`. Agent, not skill.

**Purpose:** Detect existing design system (Tailwind config, CSS variables, component library imports) and produce `.claude/design-tokens.md` for `/prototype` to read.

**Behavior:**
1. Scan for `tailwind.config.*`, `styles/tokens.*`, `theme.*`, component library imports.
2. Extract color, spacing, typography tokens.
3. Write `.claude/design-tokens.md` so `/prototype` can produce designs that respect the existing system.

**Does not integrate with Figma.** That's a separate, heavier problem. This agent makes prototypes respect the existing token system, no more.

## 4.4 — Optional: `.claude/decisions.md` (decision log)

Path: `.claude/decisions.md`. Append-only.

**Purpose:** Capture architectural and product decisions that have lasting consequences, separate from the day-to-day `progress.md` log.

**Format (ADR-style):**

```markdown
## 2026-05-11 — Chose Postgres over SQLite

**Context:** We considered SQLite for simplicity. Stack-detector flagged
the existing service uses Postgres. Migration cost would exceed simplicity
benefit.

**Decision:** Postgres.

**Consequences:** Requires connection pooling on serverless. See
`.claude/architecture.md` for pooling strategy.
```

**Written by:** `/architect` and `/plan` when they make load-bearing decisions. The skills should explicitly ask the PM "is this a decision worth logging in decisions.md?" rather than logging silently.

---

# Reference Materials

## `PATTERNS.md` — common workflow sequences

Path: project root.

Document 4–6 typical sequences without enforcing any of them. Used by `project-state-detector` to recognize patterns.

**Suggested patterns:**
1. **New SaaS feature** — `/discover` (optional) → `/prd` → `/plan` → `/build-feature` → `/check-production` → `/deploy`
2. **Brownfield refactor** — `/architect` → `/plan` → `/build-feature` (with `pattern-finder` heavy) → `/check-production`
3. **Bug sprint** — `/triage` → `/build-feature` per bug → `/check-production`
4. **Greenfield MVP** — `/start` → `/discover` → `/prd` → `/architect` → `/plan` → `/build-feature` → `/deploy`
5. **Hardening pass** — `/check-production` → `/build-feature` (fixes) → `/deploy`

State detector matches current `.claude/` state against patterns and surfaces "you look like you're following Pattern N" when confidence is high. Never forces conformance.

## Auto memory leverage (no code required)

Add a note to `DESIGN.md` or `README.md`:

> Claude Code's auto memory (v2.1.59+) accumulates implementation-level
> learnings automatically: build commands, debugging patterns, naming
> conventions, environment gotchas. Run `/memory` in any session to inspect.
>
> Auto memory captures what Claude learns about the codebase. Our
> `.claude/progress.md` captures what the PM workflow has done. They are
> complementary, not redundant.

No code change. Just documents the feature so users know it exists.

## Optional: artifact maturity frontmatter

For artifacts in `.claude/` that benefit from explicit lifecycle tracking, support an optional frontmatter:

```yaml
---
status: stub | draft | reviewed | stale | archived
last-reviewed: 2026-05-11
---
```

`project-state-detector` reads this when present, falls back to git timestamps when absent. Don't enforce it — most artifacts won't need it.

---

# Implementation Order

Build in this order. Each phase depends on the previous.

1. **Phase 1** (memory + project intelligence). Without this, everything downstream lacks the foundation it needs.
2. **Phase 2.1** (smart `/prd` routing). Cheap, high-impact.
3. **Phase 2.2** (`/discover` skill). The first selectively step-decomposed skill — get the pattern right here.
4. **Phase 3.1** (`/architect` skill). Second step-decomposed skill.
5. **Phase 3.2** (`/measure` skill). Single-file, low complexity.
6. **Phase 4** (polish skills). Order doesn't matter within this phase.

Reference materials (`PATTERNS.md`, auto memory documentation) can be added at any point.

# Acceptance Criteria

The enhancements succeed if:

1. A new PM joining the project mid-stream can run `/resume` and get oriented in under 30 seconds.
2. A PM with a vague idea is routed to `/discover` instead of bouncing off `/prd`.
3. A PM with a complete PRD is never asked to do discovery work.
4. The state detector correctly identifies project mode for at least 4 of the 5 patterns in `PATTERNS.md` based on `.claude/` state alone.
5. No existing skill is hidden, removed, or gated based on detected mode.
6. The `.claude/` directory remains the single source of truth — no new databases, daemons, or external services.
7. Every skill in `skills/` follows the pre-flight / post-flight protocol.
8. `progress.md` accumulates entries automatically as skills run; no PM has to write to it manually.

# Failure Modes to Avoid

- **Mode lock-in.** If the state detector ever blocks a skill from running, the design has failed. Soft warnings only.
- **Required step decomposition.** If any skill outside `/discover` and `/architect` gets decomposed into step files, the design has failed.
- **Forced workflow selection.** If `/start` asks "which workflow do you want," the design has failed. It captures context; it does not select a method.
- **CLAUDE.md sprawl.** If `CLAUDE.md` grows beyond ~150 lines, move content to topic files referenced from it. Long `CLAUDE.md` files reduce adherence.
- **Skill bloat.** If any single-file skill exceeds 10KB, consider whether it has earned step decomposition or whether it's just doing too much.
