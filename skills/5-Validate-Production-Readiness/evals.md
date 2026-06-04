# Evals: /check-production

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. `secret-scanner` was run (required regardless of mode)
2. `stack-detector` was run before calling `prod-readiness-auditor`
3. `codebase-classifier` result was used to calibrate severity thresholds
4. Report covers all four severity levels — or explicitly states "none found" for each missing level
5. Every finding includes a file:line citation
6. Recommended fix order is documented and accounts for dependencies, not just raw severity
7. Estimated effort is provided for every Critical and every High finding
8. No fixes were applied during the audit — report-only mode was honored
9. If `--lite` mode: results explicitly cover secrets scan, build check, and health endpoint
10. `.claude/progress.md` was updated on completion
