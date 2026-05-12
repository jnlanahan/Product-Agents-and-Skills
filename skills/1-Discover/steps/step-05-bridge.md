# Step 05 — Bridge to PRD

**Goal:** Produce a complete, structured problem statement ready to hand off to `/prd`.

## Instructions

Synthesize everything from steps 01–04 into a clean, complete discovery summary. Do not interview the user further — just synthesize.

If any section from steps 01–04 is thin or missing, note `<TBD>` and call it out at the end.

## Output

Write the final section of `.claude/discovery-notes.md`:

```markdown
## Bridge (Step 05 — PRD-ready summary)

**Problem statement (one sentence):**
<The specific problem statement from step 03>

**Customer:**
<Role, situation, named anchor if available>

**Context and trigger:**
<When the problem occurs>

**Evidence:**
<What supports that this is real>

**Stakes:**
<Why now, cost of inaction, definition of solved>

**Landscape:**
<What exists, what's missing, why existing solutions fall short>

**Scope hint:**
<Based on discovery, what's the narrowest useful solution? What's explicitly out of scope?>

**Open questions for PRD:**
<Anything still unresolved that /prd should address>
```

## State update

Update frontmatter:
- Add `bridge` to `stepsCompleted`
- Set `lastStep: bridge`
- Set `nextStep: complete`

## Handoff message

Tell the user:

> "Discovery complete. Run `/prd` — it will read `.claude/discovery-notes.md` and use your discovery as the starting point, skipping the front-door interview."
