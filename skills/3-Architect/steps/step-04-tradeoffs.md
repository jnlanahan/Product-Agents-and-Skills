# Step 04 — Tradeoffs and Rejected Alternatives

**Goal:** Make key architectural decisions explicit, with rationale and rejected alternatives documented.

## Instructions

For each significant decision identified in steps 01–03, document:
- The decision made
- Why (the driving constraint or preference)
- What was considered but rejected (and why)

Common decision categories to check:
- **Build vs. buy**: did we evaluate a third-party solution and choose to build?
- **Data storage**: why this database/ORM over alternatives?
- **Auth approach**: why this auth model?
- **Sync vs. async**: any background job or queue decisions?
- **Caching strategy**: is caching needed? At what layer?
- **Scalability assumptions**: what load are we designing for?

Ask the PM:
1. Are there any decisions we're making that you want to revisit before we commit to the architecture document?
2. Are there any decisions that feel risky or uncertain?
3. Which of these decisions, once made, would be hard to reverse?

## Output

Append to `.claude/architecture.md`:

```markdown
## Tradeoffs (Step 04)

### <Decision topic>
**Decision:** <what we chose>
**Why:** <driving constraint or preference>
**Rejected:** <what we considered but didn't choose, and why>

[repeat for each significant decision]

### Reversibility risks
<Decisions that are hard to undo, flagged for extra scrutiny>
```

## Decision log prompt

After presenting tradeoffs, ask:

> "Are any of these decisions load-bearing enough to log in `.claude/decisions.md` as an ADR? I'll record each one you confirm."

For each confirmed decision, append to `.claude/decisions.md` (create if missing):

```markdown
## <date> — <decision topic>

**Context:** <brief background>

**Decision:** <what was chosen>

**Consequences:** <downstream implications>
```

## Checkpoint

Present tradeoffs and ask: "Ready to produce the full architecture document?"
