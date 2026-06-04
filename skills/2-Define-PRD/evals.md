# Evals: /prd

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. `.claude/prd.md` was written to disk
2. All 15 required sections are present (or marked `<TBD>` with rationale in Open Questions)
3. At least 5 numbered user stories follow the As-a/I-want/so-that format
4. Functional Requirements table includes an ID column and at least 3 rows
5. Each Success Metric references a specific measurement source (e.g., "from PostHog", "from Stripe dashboard")
6. Implementation Decisions describe modules and behaviors in prose — no code snippets
7. Rollout Plan includes all three: staging environment, rollback trigger, and post-launch verification step
8. Open Questions section is present (even if empty — confirms it was not skipped)
9. Risks table includes a likelihood column AND a mitigation entry for each row
10. `.claude/progress.md` was updated on completion
