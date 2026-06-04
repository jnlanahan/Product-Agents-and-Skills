# Evals Template

Place an `evals.md` file in any skill folder where output quality is variable and worth grading.
The grading agent reads this file plus the skill's output and returns PASS/FAIL per criterion.

## When to create evals.md for a skill

- AI-generated prose (PRDs, plans, architecture docs, retrospectives) — quality varies run-to-run
- Code with security or business-critical patterns (auth, payments, AI wiring) — easy to miss a check
- Process compliance where steps can silently collapse (feature build layer discipline)

Do NOT create evals.md for:
- Purely procedural skills (deploy, setup, navigation) — the process is the eval
- Pattern-following build skills (database, email, file storage) — pattern-finder enforces quality

## Format

```markdown
# Evals: [Skill Name]

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. [Specific, checkable condition — true or false from the output alone]
2. [...]
...
(max 10)
```

## Rules for writing checks

- **Binary only.** Each check must be true or false — not "is it good?" but "does X exist?"
- **Checkable from the output alone.** The grading agent cannot run code or open the browser.
- **Specific.** "Rollout plan present" is weak. "Rollout plan includes staging, rollback trigger, and post-launch verification step" is strong.
- **Max 10.** If you have more than 10, you have too many — pick the ones that catch real regressions.

## Grading agent prompt (used by the skill's post-flight)

> You are a grading agent. You have been given:
> 1. An evals list (binary pass/fail criteria)
> 2. The output from the skill run
>
> For each criterion, return PASS or FAIL. For each FAIL, add one line of reason.
> Do not add new criteria. Do not score on a scale. Binary only.

## Example: 2-Define-PRD/evals.md

```markdown
# Evals: /prd

1. `.claude/prd.md` was written
2. All 15 required sections present (or `<TBD>` with rationale in Open Questions)
3. At least 5 numbered user stories in As-a/I-want/so-that format
4. Functional Requirements table includes an ID column and at least 3 rows
5. Each Success Metric references a measurement source
6. Implementation Decisions describe modules/behaviors in prose — no code snippets
7. Rollout Plan includes staging, rollback trigger, and post-launch verification step
8. Open Questions section is present (even if empty)
9. Risks table includes likelihood column AND mitigation for each row
10. `progress.md` was updated on completion
```
