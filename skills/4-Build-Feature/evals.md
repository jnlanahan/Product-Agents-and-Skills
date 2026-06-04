# Evals: /build-feature

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. `pattern-finder` was run before any code was written
2. At least one commit exists per layer used (schema → storage → routes → hooks → components)
3. No layer commit was made with failing tests — every committed layer was green
4. A layer handoff block was emitted after each commit with all required fields
5. No mocks used for business logic — mocks appear only at system boundaries (DB, external APIs, time)
6. End-to-end verification was performed covering at least create and read paths
7. User was prompted to run `/check-production --lite` after all slices were complete
8. If `.claude/plan.md` contains a validation contract, all assertions were checked
9. No files were modified outside the explicit scope of the feature
10. `.claude/progress.md` was updated on completion
