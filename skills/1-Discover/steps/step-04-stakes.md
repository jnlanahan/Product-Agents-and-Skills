# Step 04 — Surface Why It Matters Now

**Goal:** Understand the urgency and strategic importance. Problems that don't matter now don't get solved.

## Instructions

Ask:

1. Why solve this now? What changed recently that makes this worth building?
2. Who cares about this being solved? Is this a customer pain, a business priority, a compliance requirement?
3. What is the cost of NOT solving it — to the customer, to the business, to the team?
4. What would "solved" look like in 6 months? How would you know it worked?

Probe urgency. "We've always had this problem" is not urgency. "We're losing deals because of it" or "a regulation goes into effect in 90 days" is urgency.

If there's no urgency, surface that explicitly: "This may be a real problem but not an urgent one. That's fine to know — it affects prioritization."

## Output

Append to `.claude/discovery-notes.md`:

```markdown
## Stakes (Step 04)

**Why now:** <what changed or why this matters at this moment>
**Who cares:** <customer / business / team — and why>
**Cost of inaction:** <what happens if we don't build this>
**Definition of solved:** <what would success look like in 6 months>
```

## State update

Update frontmatter:
- Add `stakes` to `stepsCompleted`
- Set `lastStep: stakes`
- Set `nextStep: bridge`
