# Step 01 — Frame the Problem Space

**Goal:** Establish what territory we're in before trying to define the specific problem.

## Instructions

Ask the user these questions (conversationally, not as a form):

1. What is the general domain or space you're working in? (e.g., "B2B SaaS for HR teams", "consumer app for dog owners", "internal tool for logistics")
2. What prompted this idea? (saw a pain point, heard it from a user, noticed a market gap, CEO request)
3. What already exists in this space? (competitor products, internal tools, manual workarounds)
4. Is this a new product, a new feature in an existing product, or a rearchitecture of something working but broken?

Probe answers that are too vague. If the user says "it's for businesses," ask what kind. If they say "I just thought it would be useful," ask what pain or moment triggered that thought.

## Output

Append to `.claude/discovery-notes.md`:

```markdown
## Frame (Step 01)

**Domain:** <one-line description of the space>
**Prompt:** <what triggered this idea>
**Existing landscape:** <what already exists>
**Type:** new product | new feature | rearchitecture
```

## State update

Update frontmatter in `.claude/discovery-notes.md`:
- Add `frame` to `stepsCompleted`
- Set `lastStep: frame`
- Set `nextStep: customer`
