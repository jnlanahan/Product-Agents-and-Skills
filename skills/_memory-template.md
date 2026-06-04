# Memory Template

Place a `memory.md` file in a skill folder only when there is something worth staging.
Not every skill needs one. Read it before generating output; apply lessons as working constraints.

## Format

```markdown
# Memory: [Skill Name]

Lessons from past runs. Max 7 entries. Newest first.
When adding entry 8, remove the oldest.
If a pattern appears twice, promote it to SKILL.md or evals.md, then delete it here.

- [date] — [one-line lesson]
- [date] — [one-line lesson]
```

## Rules for writing entries

**Write an entry only when something is:**
- Surprising — you would have done it differently without this lesson
- Cross-run — applies every time this skill runs, not just in one project context
- Not already in SKILL.md or evals.md

**Do not write entries for:**
- Things the skill already covers
- One-off project quirks that won't generalize
- Normal run activity (this is not a log)

**Format:** One line, shorthand. Omit filler words.
- Good: `"User wants personas as job titles, not archetypes."`
- Good: `"Rollout plan section gets skipped when user is in a hurry — always generate it regardless."`
- Bad: `"The user mentioned that they prefer when the output includes detailed sections with good formatting."`

**Promotion rule:** If the same lesson appears twice across separate runs → it belongs in SKILL.md or evals.md permanently. Add it there, then delete the memory entry.

## Example: 2-Define-PRD/memory.md

```markdown
# Memory: /prd

Lessons from past runs. Max 7 entries. Newest first.
When adding entry 8, remove the oldest.
If a pattern appears twice, promote it to SKILL.md or evals.md, then delete it here.

- 2026-06-03 — User prefers personas written as job titles ("Head of Product") not archetypes ("The Strategist").
```
