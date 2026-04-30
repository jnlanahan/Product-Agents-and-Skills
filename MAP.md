# The Big Picture — Everything, Through the PDLC

Every artifact in this library — agent, skill, workflow — earns its place by serving a phase of the **Product Development Lifecycle**.

```text
1 Discover  →  2 Define  →  3 Design  →  4 Build  →  5 Validate  →  6 Deploy  →  7 Learn  ↺
```

If you only read one file, this is it. The pieces are:

- **PDLC** (7 phases) — the spine. Everything else is positioned against it.
- **Skills** (19) — slash-commands the user runs *inside* a phase.
- **Agents** (6) — read-only diagnostics that skills delegate to *during* their phase.
- **Workflows** (8) — opinionated paths *through* the phases at different rigor levels.

Source-of-truth docs: [PDLC_Phases.md](PDLC_Phases.md) · [AGENTS.md](AGENTS.md) · [WORKFLOWS.md](WORKFLOWS.md) · [GAPS.md](GAPS.md)

---

## Table of contents

- [The PDLC at a glance](#the-pdlc-at-a-glance)
- [Phase by phase — what runs, what comes out](#phase-by-phase--what-runs-what-comes-out)
- [Workflows are PDLC traversals](#workflows-are-pdlc-traversals)
- [Workflow → Skill heatmap](#workflow--skill-heatmap)
- [Skill → Agent matrix](#skill--agent-matrix)
- [Reading guide](#reading-guide)

---

## The PDLC at a glance

The 7 phases, one-line goal per phase, and which skills + agents live there. Every workflow walks this same path; they differ only in rigor and emphasis.

```mermaid
flowchart LR
    P1["1 · Discover<br/><i>Is the problem worth solving?</i><br/><br/>skills: /discover<br/>agents: pain-point-miner<br/>competitive-scanner"]
    P2["2 · Define<br/><i>What to build, how to know it worked.</i><br/><br/>skills: /prd /plan /glossary<br/>/grill-me /refactor<br/>agents: —"]
    P3["3 · Design<br/><i>How it looks, flows, is built.</i><br/><br/>skills: /prototype<br/>agents: —"]
    P4["4 · Build<br/><i>Produce a working release candidate.</i><br/><br/>skills: /setup-project /code-map<br/>/setup-database /add-auth<br/>/add-payment /add-files<br/>/add-monitoring /build-feature<br/>/migrate-from-vibe<br/>agents: stack-detector<br/>pattern-finder<br/>codebase-classifier"]
    P5["5 · Validate<br/><i>Confirm it's safe, correct, useful.</i><br/><br/>skills: /triage<br/>/check-production<br/>agents: secret-scanner<br/>dependency-currency-checker<br/>prod-readiness-auditor"]
    P6["6 · Deploy<br/><i>Get to prod, get to users.</i><br/><br/>skills: /deploy<br/>agents: secret-scanner<br/>prod-readiness-auditor"]
    P7["7 · Learn<br/><i>Did it work? What next?</i><br/><br/>skills: —<br/>agents: —"]

    P1 --> P2 --> P3 --> P4 --> P5 --> P6 --> P7
    P7 -. iterate .-> P1

    NS(["/next-steps<br/><i>cross-phase compass</i>"]):::cross
    NS -.-> P2 & P4 & P5

    classDef phase fill:#eef2ff,stroke:#4f46e5,color:#1e1b4b;
    classDef cross fill:#fff4e6,stroke:#d97706,color:#7c2d12,stroke-dasharray: 3 3;
    class P1,P2,P3,P4,P5,P6,P7 phase;
```

Phase 7 (Learn) intentionally has no skills today — it's human-led. See [GAPS.md](GAPS.md). `/next-steps` is the always-on compass that orients you wherever you are in the lifecycle.

---

## Phase by phase — what runs, what comes out

A two-line description per phase, the skills available, the agents they pull in, and the artifact you should leave with.

### 1 · Discover — *is this worth solving?*

Decide whether a real problem exists, who hurts, how big, and what already exists in the space. `/discover` orchestrates open-source research (Reddit, HN, Stack Overflow, Indie Hackers, competitor sites) and synthesizes a decision-shaped writeup.

| Item | Detail |
|---|---|
| Skills | `/discover` |
| Agents | `pain-point-miner` · `competitive-scanner` |
| Output | `.claude/discover.md` — problem statement, evidence-backed pains, competitive landscape, gap analysis, recommendation (proceed / sharpen / kill) |
| Gap | TAM/sizing math — deliberately not automated; honest numbers can't be web-scraped, see [GAPS.md](GAPS.md#discover-phase) |

### 2 · Define — *what to build & how to know it worked*

Synthesize the problem into a writeable scope: PRD, success metrics, plan, vocabulary. Pressure-test the plan before you spend a sprint on it.

| Item | Detail |
|---|---|
| Skills | `/prd` · `/plan` · `/glossary` · `/grill-me` · `/refactor` (when refactoring is the work) |
| Agents | *(none — these skills work in conversation + the codebase)* |
| Output | `.claude/prd.md` · `.claude/plan.md` · `.claude/glossary.md` · `.claude/refactor-plan.md` |

### 3 · Design — *how it looks and flows*

Make the design decisions explicit. For UI work, the prototype is the design.

| Item | Detail |
|---|---|
| Skills | `/prototype` (three clickable HTML variants — pick one) |
| Agents | *(none)* |
| Output | `prototypes/variant-A.html` · `variant-B.html` · `variant-C.html` |

### 4 · Build — *produce a release candidate*

Where most of the library lives. Scaffold, wire third-party services, ship features in TDD layers. Every implementer skill calls `stack-detector` and `pattern-finder` so new code matches local style.

| Item | Detail |
|---|---|
| Skills | `/setup-project` · `/code-map` · `/setup-database` · `/add-auth` · `/add-payment` · `/add-files` · `/add-monitoring` · `/build-feature` · `/migrate-from-vibe` |
| Agents | `stack-detector` · `pattern-finder` · `codebase-classifier` (for `/migrate-from-vibe`) |
| Output | Working code in coherent waves, one commit per layer, tests at each layer |

### 5 · Validate — *safe, correct, useful*

Catch what shouldn't ship. `/check-production` orchestrates all five validating agents in parallel; `/triage` turns findings (or user reports) into structured bug reports.

| Item | Detail |
|---|---|
| Skills | `/triage` · `/check-production` |
| Agents | `secret-scanner` · `dependency-currency-checker` · `prod-readiness-auditor` (+ `stack-detector` and `codebase-classifier` for context) |
| Output | Severity-graded readiness report (Critical/High/Medium/Low) · `.claude/bugs/<name>.md` |

### 6 · Deploy — *get to prod, get to users*

Two distinct steps: code in production, feature available to users. Plus the operational scaffolding (env, domain, SSL, runbook).

| Item | Detail |
|---|---|
| Skills | `/deploy` |
| Agents | `secret-scanner` (pre-flight gate) · `prod-readiness-auditor` (if not already run) |
| Output | Deployed app, env vars, custom domain + SSL, post-deploy smoke tests, runbook |

### 7 · Learn — *did it work?*

Compare against Define's success metrics. Decide: double down, iterate, sunset. Feed insights back into Discover.

| Item | Detail |
|---|---|
| Skills | *(none — human-led)* |
| Agents | *(none)* |
| Output | Launch review, decision, backlog updates |
| Gap | A `/post-launch-review` skill is on the wishlist — see [GAPS.md](GAPS.md#learn-phase) |

---

## Workflows are PDLC traversals

The eight workflows are not eight different lifecycles. **They all walk the same seven phases.** They differ in *which phases they spend time in* and *how heavyweight* the work is at each step.

**Heatmap — read each row left-to-right as one workflow's PDLC pass.** Darker = heavier focus.

🟥 primary focus · 🟧 standard work · 🟨 light pass · ⬜ skipped or external to the library

| Workflow | 1 Discover | 2 Define | 3 Design | 4 Build | 5 Validate | 6 Deploy | 7 Learn |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| **W1** Prototype          | 🟧 | 🟧 | 🟥 | ⬜ | 🟨 | 🟨 | 🟨 |
| **W2** Production SaaS    | 🟧 | 🟥 | 🟧 | 🟥 | 🟥 | 🟧 | 🟨 |
| **W3** Add Feature        | ⬜ | 🟧 | 🟨 | 🟥 | 🟧 | 🟧 | 🟨 |
| **W4** Migrate to Prod    | 🟨 | 🟧 | ⬜ | 🟥 | 🟥 | 🟧 | 🟨 |
| **W5** Refactor           | ⬜ | 🟧 | ⬜ | 🟥 | 🟧 | 🟨 | ⬜ |
| **W6** Bug Hotfix         | ⬜ | ⬜ | ⬜ | 🟧 | 🟥 | 🟧 | ⬜ |
| **W7** Audit & Harden     | ⬜ | ⬜ | ⬜ | 🟧 | 🟥 | 🟧 | 🟨 |
| **W8** Personal Tool      | 🟨 | 🟨 | 🟨 | 🟧 | 🟨 | 🟧 | ⬜ |

> Discover and Learn are mostly outside the library today. The intensities reflect *whether you should think about the phase*, not whether a skill runs.

---

## Workflow → Skill heatmap

Same data as [WORKFLOWS.md § heatmap](WORKFLOWS.md#workflow--skills-heatmap), kept here so this page is self-contained. **●** = always used. **○** = conditional.

| Skill | W1 Prototype | W2 SaaS | W3 Feature | W4 Migrate | W5 Refactor | W6 Hotfix | W7 Audit | W8 Personal |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `/next-steps` | | ● | ● | ● | ● | ○ | ● | ○ |
| `/setup-project` | | ● | | | | | | ● |
| `/discover` | ○ | ● | | ○ | | | | |
| `/prd` | ○ | ● | ● | ○ | | | | ○ |
| `/plan` | ○ | ● | ● | ● | ● | | | ○ |
| `/refactor` | | | ○ | | ● | ○ | ○ | |
| `/glossary` | ○ | ● | ○ | | | | | |
| `/grill-me` | ○ | ● | ○ | | ● | | | |
| `/prototype` | ● | ● | ○ | | | | | ○ |
| `/code-map` | | | ● | ● | ● | ○ | | |
| `/setup-database` | | ● | ○ | ● | | | ○ | ● |
| `/add-auth` | | ● | ○ | ○ | | | ○ | ○ |
| `/add-payment` | | ● | ○ | ○ | | | ○ | |
| `/add-files` | | ● | ○ | ○ | | | ○ | ○ |
| `/add-monitoring` | | ● | ○ | ● | | | ● | |
| `/build-feature` | | ● | ● | ○ | ● | ● | ● | ● |
| `/migrate-from-vibe` | | | | ● | | | | |
| `/triage` | | ● | ● | ● | ● | ● | ● | ○ |
| `/check-production` | | ● | ● | ● | ● | ● | ● | ○ |
| `/deploy` | ○ | ● | ● | ● | ● | ● | ● | ● |

---

## Skill → Agent matrix

Which agents each skill delegates to. **●** = primary caller. **○** = ad-hoc / conditional.

| Skill ↓ \ Agent → | `stack-detector` | `codebase-classifier` | `pattern-finder` | `secret-scanner` | `dependency-currency-checker` | `prod-readiness-auditor` | `pain-point-miner` | `competitive-scanner` |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `/next-steps` | ● | ● | | | ● | | | |
| `/setup-project` | ● | | | | | | | |
| `/discover` | | | | | | | ● | ● |
| `/setup-database` | ● | | ● | | | | | |
| `/add-auth` | ● | | ● | | | | | |
| `/add-payment` | ● | | ● | | | | | |
| `/add-files` | ● | | ● | | | | | |
| `/build-feature` | ● | | ● | | | | | |
| `/migrate-from-vibe` | ● | ● | ● | | | | | |
| `/triage` | ○ | | ○ | ○ | | | | |
| `/check-production` | ● | ● | | ● | ● | ● | | |
| `/deploy` | | | | ● | | ● | | |

Skills not listed (`/prd`, `/plan`, `/refactor`, `/glossary`, `/grill-me`, `/prototype`, `/code-map`, `/add-monitoring`) call no agents — they work in conversation.

---

## Reading guide

| You want to… | Open |
|---|---|
| Understand the lifecycle deeply | [PDLC_Phases.md](PDLC_Phases.md) |
| Pick a workflow for what you're doing | [WORKFLOWS.md](WORKFLOWS.md) |
| Look up one specific skill or agent | [AGENTS.md](AGENTS.md) |
| See what's missing from the library | [GAPS.md](GAPS.md) |
| Read the locked greenfield stack | [skills/_stack-preferences.md](skills/_stack-preferences.md) |
| See the rules for adapting to existing codebases | [skills/_adaptation-playbook.md](skills/_adaptation-playbook.md) |
| Walk through a worked example | any [workflows/Wn-*.md](workflows/) page |

---

*This page is hand-curated against [PDLC_Phases.md](PDLC_Phases.md), [AGENTS.md](AGENTS.md), and the per-workflow files. When the lifecycle understanding shifts, this page changes first.*
