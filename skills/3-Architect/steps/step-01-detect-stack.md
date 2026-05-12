# Step 01 — Detect Stack

**Goal:** Establish what technology stack is already in play before making architecture decisions.

## Instructions

Run `stack-detector` and `codebase-classifier` in parallel.

Summarize the results for the PM:

- Framework and version
- Database and ORM
- Auth provider (if any)
- Payment processor (if any)
- Monitoring (if any)
- Deploy target (if any)
- Codebase classification (greenfield / wired / vibe-coded)

Then ask:

1. Is this stack profile accurate? Any exceptions or additions I should know?
2. Are there any stack constraints I should design around? (e.g., "We can't add another dependency", "We must stay on Node 18")
3. Is the intent to extend the existing stack or introduce a new layer?

## Output

Append to `.claude/architecture.md` (create if missing):

```markdown
---
status: draft
last-reviewed: <today's date>
---

# Architecture

## Stack Profile (Step 01)

<Summary of detected stack>

**Constraints noted:**
<Any PM-specified constraints>

**Extend vs. introduce:** <extend existing | introduce new layer>
```

## Checkpoint

Present the output and ask: "Does this stack profile accurately represent what we're working with? Ready to proceed to data model?"
