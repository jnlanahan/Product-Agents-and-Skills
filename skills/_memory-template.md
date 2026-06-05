# Memory Template

Place a `memory.md` in a skill folder to record where the skill's assumptions were wrong.
Read it before generating output; apply entries as working constraints.

## When to write an entry

Write an entry when the skill assumed something that wasn't true for this run:
- Prompted for something the user already had done
- Over-explained something the user didn't need
- Missed a step that turned out to matter
- Assumed a project structure or context that wasn't there

Do not write entries for normal run activity, one-off quirks, or things already in SKILL.md.

## Format

```markdown
# Memory: /[skill-name]

Write an entry when the skill assumed something that wasn't true for this run.
Max 7 entries. Newest first. One line each.
If a pattern appears twice → add it to SKILL.md, then delete it here.

- 2026-06-04 — User already had migrations configured — skip the setup walkthrough.
- 2026-05-20 — Stack already had Stripe wired; the "install stripe" step was redundant.
```

## Hard limits

- Max 7 entries. When adding an 8th, remove the oldest.
- If the same lesson appears twice across runs → it belongs in SKILL.md permanently. Add it there, delete from memory.
- Do NOT write skill-level lessons to the global MEMORY.md.
