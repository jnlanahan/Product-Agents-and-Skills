---
name: handoff
when_to_use: "User says 'write a handoff doc', 'brief a contractor', 'package this for the team', 'external handoff', or types /handoff."
description: Use to package PRD, plan, and architecture into a single standalone document for stakeholders without codebase access — contractors, outsourced developers, executives, or design partners. Output is .claude/handoff-<feature-name>.md. Trigger on /handoff, "write a handoff doc", "brief a contractor", "package this for the team", "external handoff".
---

# /handoff

Package everything a contractor, external developer, or executive needs to understand what to build — without requiring codebase access.

## Pre-flight

- Read `.claude/prd.md` if it exists (required — no PRD means no handoff).
- Read `.claude/plan.md` if it exists.
- Read `.claude/architecture.md` if it exists.
- Read `.claude/measurement.md` if it exists.
- Read `.claude/context.md` if present.
- Read `.claude/progress.md` (last 5 entries).
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block).

## Post-flight

- Append to `.claude/progress.md`: timestamp, `/handoff`, output path, key decisions, suggested next step.
- If `.claude/progress.md` is missing, create it with a header first.

## When to Use

- Briefing an external contractor or agency
- Presenting to an executive who won't read code
- Handing off to a developer who is joining mid-project
- Creating a reference doc for a design partner

## When NOT to Use

- Internal handoff between engineers on the same team → use `.claude/plan.md` directly
- Live product documentation → wrong tool; use a wiki or README

## Procedure

### Step 1: Confirm feature name

Ask: "What is the name of the feature or project this handoff is for?" Use this as the filename slug (`handoff-<slug>.md`).

### Step 2: Synthesize the handoff document

Assemble from all available `.claude/` artifacts. Do not expose internal file paths or codebase-specific jargon. Write for a smart reader with no repo access.

### Step 3: Write .claude/handoff-<slug>.md

```markdown
---
prepared: <today's date>
feature: <feature name>
status: draft
---

# Handoff: <Feature Name>

## What we're building
<2–3 paragraph plain-language description. What is it? Why does it matter? Who uses it?>

## Problem statement
<The validated problem this feature solves, from the PRD>

## Customer
<Who this is for — role, context, what they do today without this feature>

## Success criteria
<How we'll know this worked. Metrics and targets from .claude/measurement.md or PRD>

## Scope: In

<Bulleted list of what is included in this build>

## Scope: Out

<Bulleted list of explicit exclusions — things that might seem in-scope but aren't>

## Implementation overview

<Non-technical summary of the major components or layers. If an architecture doc exists, summarize the key decisions here in plain language>

## Acceptance criteria

<PM-readable validation contracts. Each item should be verifiable by a non-engineer:>

- [ ] <User can do X and sees Y result>
- [ ] <When Z happens, the system does W>
- [ ] <Edge case: if A, then B>

## Key decisions

<Important choices already made and why — so the contractor doesn't re-litigate them>

## Open questions

<Anything still TBD that the contractor needs input on before starting>

## Contacts and approvals

| Role | Name | Approves |
|---|---|---|
| PM | <name> | Requirements, scope changes |
| Tech lead | <name> | Architecture, major impl decisions |
| Design | <name> | UI/UX decisions |

## Reference materials

- PRD: `.claude/prd.md`
- Implementation plan: `.claude/plan.md`
- Architecture: `.claude/architecture.md` (if available)
```

## Constraints

- Write for a reader with no codebase access — no file paths, no internal jargon.
- Acceptance criteria must be PM-readable, not engineer-readable. No code. No API specs.
- If key inputs are missing (`prd.md` doesn't exist), stop and tell the user: "I need a PRD before I can produce a handoff doc. Run `/prd` first."
