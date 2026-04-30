# Product Agents & Skills

> A curated library of [Claude Code](https://claude.com/claude-code) **agents** and **skills** that drive a full Product Development Lifecycle — from prototyping a new idea to shipping a production-grade SaaS, with eight named workflows that compose them.

[![PDLC](https://img.shields.io/badge/PDLC-7%20phases-1f6feb)](PDLC_Phases.md)
[![Agents](https://img.shields.io/badge/agents-6-2da44e)](AGENTS.md#agents-read-only-diagnostics)
[![Skills](https://img.shields.io/badge/skills-19-2da44e)](AGENTS.md#skills-by-pdlc-phase)
[![Workflows](https://img.shields.io/badge/workflows-8-orange)](WORKFLOWS.md)

---

## What this is

Building product with AI assistants gets messy fast. You end up re-explaining your stack to every new chat, hand-orchestrating "now do auth, now do payments, now check for secrets," and rediscovering production gotchas the hard way. This repo packages that knowledge into **reusable, scoped, well-described agents and skills** so the assistant gets it right the first time.

It's organized around a **7-phase Product Development Lifecycle** — Discover → Define → Design → Build → Validate → Deploy → Learn — and offers **8 named workflows** that compose the agents and skills differently depending on what you're doing:

| You want to… | Workflow |
|---|---|
| Validate an idea with a clickable mockup | [**W1 — Prototype a New Idea**](workflows/W1-prototype-new-idea.md) |
| Ship a real SaaS with auth, payments, monitoring | [**W2 — Build a Production-Grade SaaS Product**](workflows/W2-production-saas.md) |
| Add a feature to a live product | [**W3 — Add a Feature to an Existing Product**](workflows/W3-add-feature.md) |
| Rescue a Replit / V0 / Lovable / Bolt MVP | [**W4 — Migrate a Prototype to Production**](workflows/W4-migrate-to-production.md) |
| Pay down debt without changing behavior | [**W5 — Refactor & Modernize**](workflows/W5-refactor-modernize.md) |
| Triage and fix a production bug | [**W6 — Fix a Production Bug**](workflows/W6-fix-production-bug.md) |
| Audit an existing app and harden it | [**W7 — Audit & Harden for Production Launch**](workflows/W7-audit-harden.md) |
| Build something just for yourself | [**W8 — Build a Personal-Use Tool**](workflows/W8-personal-tool.md) |

---

## Repository structure

```
.
├── README.md                  ← you are here
├── AGENTS.md                  ← index of every agent + skill
├── WORKFLOWS.md               ← index of the 8 workflows
├── GAPS.md                    ← known coverage gaps + future work
├── PDLC_Phases.md             ← the 7-phase lifecycle this is organized around
├── CLAUDE.md                  ← orientation for any AI agent opening the repo cold
├── LICENSE
├── settings.snippet.json      ← optional Claude Code hook config
│
├── agents/                    ← 6 read-only diagnostic agents
│   ├── stack-detector.md
│   ├── codebase-classifier.md
│   ├── pattern-finder.md
│   ├── prod-readiness-auditor.md
│   ├── secret-scanner.md
│   └── dependency-currency-checker.md
│
├── skills/                    ← 19 conversational skills (slash-commands)
│   ├── _stack-preferences.md  ← locked greenfield stack
│   ├── _adaptation-playbook.md← rules for adapting to existing codebases
│   ├── 0-Always-Next-Steps/
│   ├── 0-Setup-Project/
│   ├── 2-Define-PRD/
│   ├── 2-Define-Plan/
│   ├── 2-Define-Refactor/
│   ├── 2-Define-Glossary/
│   ├── 2-Define-Grill-Me/
│   ├── 3-Design-Prototype/
│   ├── 4-Build-Code-Map/
│   ├── 4-Build-Database/
│   ├── 4-Build-Auth/
│   ├── 4-Build-Payments/
│   ├── 4-Build-File-Storage/
│   ├── 4-Build-Monitoring/
│   ├── 4-Build-Feature/
│   ├── 4-Build-Migrate-From-Vibe/
│   ├── 5-Validate-Triage/
│   ├── 5-Validate-Production-Readiness/
│   └── 6-Deploy/
│
└── workflows/                 ← per-workflow pages with diagrams
    ├── W1-prototype-new-idea.md
    ├── W2-production-saas.md
    ├── W3-add-feature.md
    ├── W4-migrate-to-production.md
    ├── W5-refactor-modernize.md
    ├── W6-fix-production-bug.md
    ├── W7-audit-harden.md
    └── W8-personal-tool.md
```

---

## Quick start

### Use the agents and skills locally

These agents and skills are designed to be installed under your **global** Claude Code config (`~/.claude/`), not project-scoped, so you have them available across every project.

```bash
# clone the repo somewhere
git clone https://github.com/<you>/product-agents-and-skills.git
cd product-agents-and-skills

# copy agents and skills into your global Claude Code config
mkdir -p ~/.claude/agents ~/.claude/skills
cp agents/*.md ~/.claude/agents/
cp -r skills/* ~/.claude/skills/

# (optional) merge the hook config
# open settings.snippet.json and merge into ~/.claude/settings.json
```

Then in any project, in Claude Code, the slash-commands are available:

```
/next-steps      ← always a good first command
/prd             ← synthesize a PRD
/prototype       ← three clickable HTML mockups
/setup-project   ← scaffold a new SaaS
/check-production ← deep readiness audit
/deploy          ← ship to prod
… and 13 more
```

### Pick a workflow

Start with [WORKFLOWS.md](WORKFLOWS.md) and find the row that matches what you're doing. Each workflow has a diagram, an ordered skill sequence, and an example walkthrough.

If you're not sure, run `/next-steps` in your project — it tells you where you're at and what to run next.

---

## Design principles

Every agent and skill follows the same rubric ([10 best practices](AGENTS.md#design-principles)):

1. **Right mechanism** — subagents for delegated diagnostic work, skills for repeating procedural workflows
2. **Action-oriented descriptions** — every agent's `description` is a delegation trigger, not a summary
3. **Tools scoped explicitly** — least-privilege; reviewers get `Read, Grep, Glob`; only one agent has network access
4. **One job per agent** — six small specialists, not one mega-agent
5. **Stateless invocations** — every needed input is passed in the prompt
6. **Behavior in the body** — concrete constraints live in the system prompt, not the description
7. **Structured output** — `STACK PROFILE`, `PATTERN`, `SECRET SCAN REPORT` — every agent returns a labeled block
8. **Skills for procedural knowledge that repeats** — anything you find yourself re-explaining belongs in a skill
9. **Evaluate before optimizing** — these were built against real project failures, not imagined ones
10. **Read-only first** — all six agents are read-only by design; write capability lives in skills with the user in the loop

---

## What's NOT covered

The library is opinionated about its scope. See [GAPS.md](GAPS.md) for the full backlog of missing skills (email, AI, accessibility, rollback, post-launch review, and more) and the principles behind what we deliberately don't add (no auth-provider migrations, no auto-fix, no unattended refactors).

---

## Contributing

If a gap is biting you:

1. Read the relevant section of [GAPS.md](GAPS.md).
2. Draft the smallest possible skill (`SKILL.md` with action-oriented frontmatter).
3. List the agents it calls; if you need a new diagnostic, draft it as a read-only agent first.
4. Add it to [AGENTS.md](AGENTS.md), update affected [workflows](workflows/), remove its row from `GAPS.md`.
5. Verify on a real project before merging.

---

## License

[MIT](LICENSE) — use, fork, modify freely.

---

## Acknowledgments

Built around the [Claude Code](https://claude.com/claude-code) agent and skill primitives, organized against [Anthropic's agent design best practices](https://docs.anthropic.com/en/docs/claude-code/sub-agents).
