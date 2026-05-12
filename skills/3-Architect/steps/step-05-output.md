# Step 05 — Assemble Architecture Document

**Goal:** Produce a complete, reviewable `.claude/architecture.md` from all prior step outputs.

## Instructions

Assemble all sections produced in steps 01–04 into a single cohesive document. Do not repeat questions — synthesize.

Add any sections that are missing or thin by inferring from the PRD and conversation context. Where inference was used, note `[inferred — please review]`.

## Output

Write `.claude/architecture.md` (overwrite the draft from earlier steps with the complete version):

```markdown
---
status: draft
last-reviewed: <today's date>
---

# Architecture — <feature or product name>

## Summary
<3–5 sentence overview of the architectural approach>

## Stack
<From step 01>

## Data Model
<From step 02>

## Integrations
<From step 03>

## Key Decisions and Tradeoffs
<From step 04>

## Open Questions
<Any unresolved items that /plan or /build-feature will need to address>

## Next Steps
- Run `/plan` to decompose this architecture into vertical implementation slices.
- Run `/measure` to define success metrics aligned with this architecture.
```

## Completion message

Tell the user:

> "Architecture document complete at `.claude/architecture.md`. Suggested next step: `/plan` to turn this into implementation slices, or `/measure` to define the telemetry strategy."

Update the architecture document's frontmatter:
- `status: draft` (the PM should change to `reviewed` after reviewing)
