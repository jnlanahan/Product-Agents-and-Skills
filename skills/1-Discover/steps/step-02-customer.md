# Step 02 — Articulate the Specific Customer

**Goal:** Get from "businesses" or "users" to a named, specific person with a concrete situation.

## Instructions

Push for specificity. Generic customers produce generic products.

Ask:

1. Who specifically has this problem? Name a role, a company type, a life situation.
2. Do you have a real person in mind — a specific user you've talked to, observed, or heard from?
3. What is that person doing right now to solve this problem (or not solving it at all)?
4. What makes this person's situation different from a similar person who doesn't have this problem?

If the user names a real person or company, excellent — use them as the concrete anchor. If not, help them construct a specific persona based on their answers. Resist staying at "any knowledge worker" or "enterprise companies."

## Output

Append to `.claude/discovery-notes.md`:

```markdown
## Customer (Step 02)

**Who:** <specific role / person type / company type>
**Named anchor:** <real person or persona name, if available>
**Current behavior:** <what they do now to address the problem>
**Differentiator:** <what distinguishes this customer from one who doesn't have the problem>
```

## State update

Update frontmatter:
- Add `customer` to `stepsCompleted`
- Set `lastStep: customer`
- Set `nextStep: problem`
