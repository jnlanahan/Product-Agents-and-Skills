# Step 03 — Sharpen the Problem Statement

**Goal:** Produce a precise, falsifiable problem statement that a product can address.

## Instructions

A good problem statement:
- Names the specific customer (from step 02)
- Describes the situation in which the problem occurs
- States the consequence or cost of the problem not being solved
- Is falsifiable (you could imagine evidence that disproves it)

Ask:

1. In one sentence, what is the problem that [customer from step 02] has?
2. When does this problem occur? In what context or workflow?
3. What happens when this problem is not solved? What's the consequence?
4. How do you know this is a real problem? What evidence do you have?

Probe the "evidence" question hard. "I think" is not evidence. "Three people told me" or "I watched someone fail to do this" is evidence.

Help the user iterate on their one-sentence problem statement until it is:
- Specific (not "people have trouble with X")
- Situated (has a context or trigger)
- Consequential (names the cost)

## Output

Append to `.claude/discovery-notes.md`:

```markdown
## Problem (Step 03)

**Statement:** <one-sentence problem statement>
**Context:** <when / in what workflow does this occur>
**Consequence:** <what happens if unsolved>
**Evidence:** <what data or observation supports this>
```

## State update

Update frontmatter:
- Add `problem` to `stepsCompleted`
- Set `lastStep: problem`
- Set `nextStep: stakes`
