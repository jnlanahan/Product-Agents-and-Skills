# Workflows

Eight named paths through the [PDLC](PDLC_Phases.md), each composed from the [agents and skills](AGENTS.md) in this repo. They differ in **intent**, **rigor**, and **which phases get the most weight** — not in lifecycle. Every workflow walks the same seven phases (Discover → Define → Design → Build → Validate → Deploy → Learn); the lighter ones spend less time in some and more in others, but they don't *skip* the thinking the lifecycle expects.

For the one-page big-picture view of how phases, skills, and agents connect, see [MAP.md](MAP.md).

---

## How to choose a workflow

Match what you're doing to the **Pick when** column. If two fit, pick the heavier one — easier to skip rigor than to retrofit it.

| # | Workflow | Pick when |
|---|---|---|
| **W1** | [Prototype a New Idea](workflows/W1-prototype-new-idea.md) | You want a clickable mockup in hours to validate a concept. Throwaway. |
| **W2** | [Build a Production-Grade SaaS Product](workflows/W2-production-saas.md) | Validated pain point → real SaaS with auth, payments, monitoring, deploy. |
| **W3** | [Add a Feature to an Existing Product](workflows/W3-add-feature.md) | Backlog feature on a live codebase. Minimal disruption. |
| **W4** | [Migrate a Prototype to Production](workflows/W4-migrate-to-production.md) | Rescue a Replit / V0 / Lovable / Bolt / ChatGPT MVP. |
| **W5** | [Refactor & Modernize an Existing Codebase](workflows/W5-refactor-modernize.md) | Pay down debt, deepen modules — no behavior change. |
| **W6** | [Fix a Production Bug](workflows/W6-fix-production-bug.md) | Customer-reported bug or PagerDuty page. |
| **W7** | [Audit & Harden for Production Launch](workflows/W7-audit-harden.md) | Existing app "works" but hasn't been scrutinized. |
| **W8** | [Build a Personal-Use Tool](workflows/W8-personal-tool.md) | Solo / private tool. No Stripe, no full auth, no Sentry/PostHog. |

---

## Master diagram

All eight workflows mapped onto the seven PDLC phases. Solid arrows = primary flow. Dashed arrows = optional / conditional.

```mermaid
flowchart LR
    subgraph PDLC["PDLC Phases"]
        direction LR
        D1["1. Discover"]
        D2["2. Define"]
        D3["3. Design"]
        D4["4. Build"]
        D5["5. Validate"]
        D6["6. Deploy"]
        D7["7. Learn"]
        D1 --> D2 --> D3 --> D4 --> D5 --> D6 --> D7
    end

    W1["W1 · Prototype a New Idea"] --> D2
    W1 --> D3
    W1 -.-> D5
    W1 -.-> D7

    W2["W2 · Production-Grade SaaS"] --> D2
    W2 --> D3
    W2 --> D4
    W2 --> D5
    W2 --> D6

    W3["W3 · Add a Feature"] --> D2
    W3 -.-> D3
    W3 --> D4
    W3 --> D5
    W3 --> D6

    W4["W4 · Migrate to Production"] --> D2
    W4 --> D4
    W4 --> D5
    W4 --> D6

    W5["W5 · Refactor"] --> D2
    W5 --> D4
    W5 --> D5
    W5 -.-> D6

    W6["W6 · Bug Hotfix"] --> D4
    W6 --> D5
    W6 --> D6

    W7["W7 · Audit & Harden"] --> D5
    W7 --> D4
    W7 --> D6

    W8["W8 · Personal Tool"] -.-> D2
    W8 -.-> D3
    W8 --> D4
    W8 -.-> D5
    W8 --> D6
```

> Mermaid renders inline on GitHub. If you're reading the source, paste the block into [mermaid.live](https://mermaid.live) for an interactive view.

---

## Workflow → skills heatmap

Which skills each workflow touches. **●** = always used. **○** = conditional / optional. Blank = not used.

| Skill | W1 | W2 | W3 | W4 | W5 | W6 | W7 | W8 |
|---|:-:|:-:|:-:|:-:|:-:|:-:|:-:|:-:|
| `/start` | ○ | ● | ○ | ○ | ○ | | | ○ |
| `/next` | | ● | ● | ● | ● | ○ | ● | ○ |
| `/resume` | ○ | ● | ● | ● | ● | ○ | ● | ○ |
| `/workflow` | ○ | ○ | ○ | ○ | ○ | ○ | ○ | ○ |
| `/setup-project` | | ● | | | | | | ● |
| `/discover` | ○ | ● | ○ | | | | | |
| `/prd` | ○ | ● | ● | ○ | | | | ○ |
| `/plan` | ○ | ● | ● | ● | ● | | | ○ |
| `/measure` | | ● | ○ | | | | | |
| `/refactor` | | | ○ | | ● | ○ | ○ | |
| `/glossary` | ○ | ● | ○ | | | | | |
| `/grill-me` | ○ | ● | ○ | | ● | | | |
| `/architect` | | ● | ○ | | | | | |
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
| `/uat` | | ● | ● | ● | ○ | | ○ | |
| `/accessibility` | | ● | ○ | ● | | | ● | |
| `/check-production` | | ● | ● | ● | ● | ● | ● | ○ |
| `/feature-flag` | | ● | ● | ○ | | | | |
| `/rollback` | | ○ | ○ | ○ | | ● | | |
| `/runbook` | | ● | ○ | ● | | | | |
| `/deploy` | ○ | ● | ● | ● | ● | ● | ● | ● |
| `/handoff` | | ● | ○ | ○ | | | | |
| `/post-launch-review` | | ● | ○ | ● | | | | |
| `/postmortem` | | ○ | ○ | | | ○ | ○ | |

---

## Read each workflow

Each workflow has its own page with **when to use**, **PDLC mapping**, **ordered skill sequence**, **flow diagram**, **agents called**, **gaps surfaced**, and an **example walkthrough**:

- [W1 — Prototype a New Idea](workflows/W1-prototype-new-idea.md)
- [W2 — Build a Production-Grade SaaS Product](workflows/W2-production-saas.md)
- [W3 — Add a Feature to an Existing Product](workflows/W3-add-feature.md)
- [W4 — Migrate a Prototype to Production](workflows/W4-migrate-to-production.md)
- [W5 — Refactor & Modernize an Existing Codebase](workflows/W5-refactor-modernize.md)
- [W6 — Fix a Production Bug](workflows/W6-fix-production-bug.md)
- [W7 — Audit & Harden for Production Launch](workflows/W7-audit-harden.md)
- [W8 — Build a Personal-Use Tool](workflows/W8-personal-tool.md)
