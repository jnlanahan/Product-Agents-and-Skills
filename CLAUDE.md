# CLAUDE.md — Repo Orientation

You are working in **Product Agents & Skills** — a curated library of Claude Code agents and skills organized around a 7-phase Product Development Lifecycle (PDLC).

## What this repo IS

A library you publish for others (and yourself) to install into their global Claude Code config (`~/.claude/agents/`, `~/.claude/skills/`). It is **not** an application — there is no `package.json`, no app code to run, no tests to pass.

## What it contains

- **8 agents** in [agents/](agents/) — read-only diagnostic helpers (stack-detector, codebase-classifier, pattern-finder, prod-readiness-auditor, secret-scanner, dependency-currency-checker, project-state-detector, design-tokens-detector).
- **27 skills** in [skills/](skills/) — conversational slash-command workflows organized by PDLC phase (`0-Setup-*`, `1-Discover-*`, `2-Define-*`, `3-*`, `4-Build-*`, `5-Validate-*`, `6-Deploy-*`, `7-Learn-*`).
- **8 workflows** in [workflows/](workflows/) — named end-to-end paths through the agents and skills for different intents.
- **Reference docs** — [PDLC_Phases.md](PDLC_Phases.md), [skills/_stack-preferences.md](skills/_stack-preferences.md), [skills/_adaptation-playbook.md](skills/_adaptation-playbook.md).
- **Indexes** — [MAP.md](MAP.md) (one-page big-picture map of workflows × skills × agents), [AGENTS.md](AGENTS.md) (everything), [WORKFLOWS.md](WORKFLOWS.md) (composed paths), [GAPS.md](GAPS.md) (missing pieces).

## Where to start when working in this repo

| Task | Read first |
|---|---|
| Adding a new skill | [AGENTS.md](AGENTS.md) (existing patterns) → [GAPS.md](GAPS.md) (is it already on the list?) → [skills/_adaptation-playbook.md](skills/_adaptation-playbook.md) |
| Adding a new agent | [AGENTS.md](AGENTS.md) "Agents" section → existing read-only agents in [agents/](agents/) for shape |
| Adding or editing a workflow | [WORKFLOWS.md](WORKFLOWS.md) → relevant `workflows/Wn-*.md` |
| Updating the index | [AGENTS.md](AGENTS.md) — keep frontmatter `description` and the index entry in sync |
| Closing a gap | [GAPS.md](GAPS.md) — when you ship the skill, remove its row |
| Showing how everything fits together | [MAP.md](MAP.md) — diagrams + cross-reference tables |

## Conventions

1. **Skills go in `skills/<phase>-<area>-<name>/SKILL.md`.** Phase numbers match PDLC stages: `0` setup/navigation, `1` discover, `2` define, `3` design, `4` build, `5` validate, `6` deploy, `7` learn.
2. **Agent frontmatter must declare `tools:`** — least privilege, no defaults.
3. **Descriptions are delegation triggers** — start with "MUST BE USED for X" or "Use when Y to produce Z." Not summaries.
4. **All agents in this repo are read-only** — no Write/Edit tools. If you propose a write-capable agent, justify it against best practice #10 in [AGENTS.md](AGENTS.md#design-principles).
5. **Every workflow doc contains:** *when to use*, *PDLC mapping*, *skill sequence*, *Mermaid diagram*, *agents called*, *gaps surfaced*, *example walkthrough*.

6. **Model tiers for agents**: `haiku` = fast structured read-only agents called freely per skill invocation (stack-detector, codebase-classifier, pattern-finder). `sonnet` = deep analysis agents called once per workflow (prod-readiness-auditor, secret-scanner, etc.). Skills themselves carry no model field; users invoke planning sessions with `/model opus` for best reasoning quality on `/prd` and `/plan`.
7. **Keep SKILL.md concise**: Core skill instructions must stay under 5,000 words. Move large reference material (templates, examples, API patterns) into a `references/` subfolder and link from the skill with `→ See [file.md](references/file.md)`. This prevents context window bloat when many skills are loaded simultaneously.

## What NOT to do

- Don't add migration skills between auth providers or payment processors. See [GAPS.md](GAPS.md#principles-for-what-we-deliberately-do-not-add).
- Don't add unattended `auto-fix` modes. Best practice #10 — read-only first; humans approve writes.
- Don't bypass the indexes. If you add a skill or agent, update [AGENTS.md](AGENTS.md) and any affected [workflow files](workflows/) in the same change.

## Memory protocol

Every skill in this project follows this discipline:

1. Read `.claude/progress.md` before beginning work. It is the project's
   narrative log — what was done, what's next, what's blocked.
2. Read any artifacts listed as inputs (`.claude/prd.md`, `.claude/plan.md`,
   `.claude/architecture.md`, `.claude/context.md`, etc.) before generating output.
3. On completion, append a 3–5 line entry to `.claude/progress.md`:
   timestamp, skill name, output path, key decisions, suggested next step.
4. Read `memory.md` in this skill's folder (if it exists) **before generating output**.
   Apply the lessons as working constraints — they override default behavior.
5. After delivering output, if `evals.md` exists in this skill's folder:
   Spawn a fresh grading agent (clean context). Give it the evals list and the output just produced.
   Grading agent instruction: "Return PASS or FAIL for each criterion. For FAILs, one-line reason only. Do not add criteria."
   Present the pass/fail table to the user. FAILs are suggestions — the user decides whether to correct.
   Do NOT auto-apply corrections or re-run the skill.
6. After the grading pass (or if no evals.md exists), always surface 1–3 specific enhancement
   suggestions for this skill based on what was ambiguous, hard, or surprising during the run.
   These are observations — the user decides whether to update evals.md, memory.md, or SKILL.md.

Assume the context window resets at any moment. Anything not written to
`.claude/` is lost. Skills must not rely on conversational memory to carry
state forward.

If `.claude/progress.md` does not exist, create it with a header before
appending the first entry.

## Skill Memory Rules

`memory.md` is a staging area for things not yet codified in the skill. It is not a run log.

Write an entry ONLY when something is:
- **Surprising** — you would have done it differently without this lesson
- **Cross-run** — applies every time this skill runs, not just in this context
- **Not already in SKILL.md or evals.md**

Format: one line, shorthand. Example: `"User wants personas as job titles, not archetypes."`

Hard limits:
- Max 7 entries per skill
- If a pattern appears in memory **twice** → promote it to SKILL.md or evals.md, then delete it from memory
- If something is already in SKILL.md → do not put it in memory
- Not every skill needs a memory file. Only create one when there is something worth staging.

Do NOT write skill-level lessons to the global MEMORY.md.

## Project context

Persistent project context lives at `.claude/context.md`. Read it at session
start. It contains stack choices, conventions, constraints, and stakeholder
notes that don't change frequently.
