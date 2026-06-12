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
11. If any secret-type key sits behind a client-exposed env prefix (`NEXT_PUBLIC_`/`VITE_`/`REACT_APP_`/`EXPO_PUBLIC_`/`PUBLIC_`/`NUXT_PUBLIC_`), it is reported as Critical — or the report states no client-bundled secrets were found
12. If the app uses Supabase or Firebase/Firestore, the report addresses data-access control (RLS enabled + scoped policies, or Firestore rules not in open/test mode) — or states it was checked and is sound
13. If the app has a client UI that fetches data, the report addresses client-side failure handling (error states / error boundary, not blank-screen-on-failed-request) — or states it was checked and is sound
