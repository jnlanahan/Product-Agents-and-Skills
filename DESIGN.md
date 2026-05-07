# Design Philosophy & Engineering Standards

**Product Agents & Skills** is a curated library of Claude Code agents and skills built around a 7-phase Product Development Lifecycle. This document explains the deliberate design decisions that make it work — and what makes it different from other AI-assisted development workflows.

---

## The Core Idea: Mini-Workflows, Not Monoliths

Most AI agent setups for product development are monolithic: one giant prompt or one all-knowing agent that tries to take you from "idea" to "deployed app" in a single run. These fail in practice because a single long context loses coherence, one agent has no checks on its own decisions, and there's no way to enter the workflow at the step that actually matches where you are.

This library is built differently: **8 named mini-workflows composed from 27 independent, interchangeable tools** — 8 read-only diagnostic agents and 19 conversational skills. Each tool does one job well. You enter the lifecycle at the step that matches your current situation, not at the beginning every time.

| | Monolithic agent | This library |
|---|---|---|
| Entry point | Always "start here" | Any PDLC phase |
| Context window | One long conversation | Fresh context per agent |
| Failure recovery | Restart everything | Fix one skill, rerun it |
| Adaptability | Fixed behavior | Detect-then-adapt per project |
| Human control | Batch approval at end | Decision gate at each skill |

---

## Key Design Highlights

### Read-Only Agents, Write-Capable Skills

All 8 agents are **strictly read-only** (`Read, Grep, Glob` only). They detect, classify, and report. They never write. Skills call them before deciding what to do — detect first, adapt second, write last. This eliminates a whole class of agent mistakes: an agent cannot break what it is analyzing.

Write capability lives exclusively in skills, where the human is always in the loop. No skill has an auto-fix mode. Every write operation follows the pattern: detect → propose → wait for approval → execute.

### Delegation Triggers, Not Descriptions

Every agent and skill description is written as a **delegation trigger** — the signal the model uses to decide *when* to invoke the tool, not a summary of what it does:

> ✗ "Generates a production readiness report."  
> ✓ "MUST BE USED by `/check-production` to perform a comprehensive 9-area audit before any production deploy."

Negative triggers are equally important. Ambiguous skills include explicit "Do NOT use for..." clauses so the model routes to the right tool rather than the nearest-sounding one.

### Model Assignment by Cognitive Demand

Not every task deserves the same model. Fast, structured file-scanning agents (stack-detector, codebase-classifier, pattern-finder) run on `haiku` — they need speed and cheapness, not deep reasoning, and they're called freely on every skill invocation. Heavy analysis agents (prod-readiness-auditor, secret-scanner, dependency-currency-checker) run on `sonnet`. Planning sessions with `/prd` and `/plan` benefit from `/model opus`.

Matching model tier to actual cognitive demand makes the library faster and cheaper to run without sacrificing quality where it counts.

### Structured Output Blocks

Every agent returns a labeled, parseable block:

```
STACK PROFILE            stack-detector
CLASSIFICATION           codebase-classifier
PATTERN                  pattern-finder
PRODUCTION READINESS     prod-readiness-auditor
SECRET SCAN REPORT       secret-scanner
CURRENCY REPORT          dependency-currency-checker
```

Calling skills read labeled sections, not free-form prose. Agent-to-skill communication is deterministic regardless of which underlying model handles the call.

### The Three-Class Codebase Model

Every `add-*` and `build-*` skill adapts based on the codebase classification (`greenfield` / `wired` / `vibe-coded`) returned by `codebase-classifier`. The adaptation rules live in [`_adaptation-playbook.md`](skills/_adaptation-playbook.md) — one place, not scattered across individual skills. A vibe-coded app gets handled carefully everywhere, not just in the skills the author happened to test on one.

### Skill Composition

`/check-production` composes 5 agents in parallel, synthesizes their outputs, and produces one unified report. `/deploy` runs `secret-scanner` as a hard gate — not a warning, a stop — before any deploy action proceeds. `/plan` feeds directly into `/build-feature` via `.claude/plan.md`.

Skills are designed to be the endpoint of one workflow and the starting point of another. Adding a new validation concern means adding one agent and wiring it into the relevant orchestrators, not touching every skill.

### Validation Contracts Before Implementation

`/plan` outputs two things: an implementation plan and a **Validation Contract** — a table of observable assertions written before any code is written. The contract describes what success looks like independently of implementation decisions, so it can be checked by a fresh reviewer at the end without creator bias.

After all slices are implemented, the contract is walked through row-by-row without consulting the implementation path. A contract violation means the feature is not done — not a test to explain away.

### Creator-Verifier Separation

The agent that builds a feature should not be the one that certifies it complete. `/build-feature` ends with an explicit independent verification step: the user runs `/check-production --lite`, which invokes `prod-readiness-auditor` with no memory of the build context. Any issues the implementer explained away get flagged fresh.

This is Design Principle #11: never let the implementer self-certify.

### Layer Handoffs in Feature Builds

`/build-feature` emits a structured `LAYER HANDOFF` block after each committed layer, recording: layer completed, files changed, tests passing, build clean, issues found, what comes next. The skill does not proceed to the next layer if tests are failing or the build is broken.

This enforces "green at each layer" as a hard constraint, not an aspiration, and prevents context drift across long feature builds.

### Vertical Slice TDD as the Default

`/plan` breaks work into vertical slices — tracer bullets through every layer, each demoable end-to-end — and assigns a TDD strategy per slice. `/build-feature` enforces one-test-per-impl and one-commit-per-layer.

This is not standard AI coding behavior. Most AI tools write all the code first and tests last (or never). The enforced vertical slice cycle prevents the common failure mode: code that compiles but doesn't behave.

### Output Caps on Heavy Agents

`prod-readiness-auditor` caps at 20 findings (Critical first); excess is noted with a count and an instruction for targeted re-runs. `dependency-currency-checker` caps at 10 dependencies and ~150 lines. Unbounded agent output is a context window tax on every subsequent step in the session.

### Parallel Reads, Serial Writes

Read-only agent calls run in parallel within skills — `/plan` runs `stack-detector` and `pattern-finder` simultaneously. Write operations are always sequential and always gate on user approval. This pattern is enforced structurally: agents have no write tools, so parallelizing them is always safe by design.

---

## How This Compares to Other PM Agent Setups

Most AI product management tools fall into one of two categories:

**Document generators** write PRDs, user stories, or roadmaps from a prompt. They're one-shot, stateless, and disconnected from the codebase or build process. They produce documents; they don't drive delivery.

**Monolithic "build me an app" agents** try to take a one-sentence description and produce a deployed application. These collapse under real-world complexity: inconsistent codebases, incremental features, production constraints, and multi-step decisions that require human judgment.

This library occupies a different position:

- **Codebase-aware at every step** — every skill that writes code first reads the project's actual patterns, stack, and classification. Nothing is generated from assumptions.
- **PDLC-spanning** — 7 phases from Define through Learn. Retros, postmortems, UAT, and rollback runbooks are first-class citizens, not afterthoughts.
- **Human-in-the-loop by design** — no auto-fix, no unattended execution. Every write gate requires approval. The AI moves fast; the human stays in control.
- **Composable, not prescriptive** — you don't follow the workflow; you assemble the tools you need. A bug fix uses `/triage`. A new SaaS uses the W2 workflow. A prototype idea uses just `/prd` + `/prototype`. Enter anywhere.
- **Failure-explicit** — every skill documents what to do when it goes wrong. Designed for real projects, where things go wrong regularly.

---

*See [WORKFLOWS.md](WORKFLOWS.md) for the 8 named workflow paths. See [AGENTS.md](AGENTS.md) for the full skill and agent index with design principles.*
