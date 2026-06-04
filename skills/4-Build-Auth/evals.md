# Evals: /add-auth

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. `pattern-finder` was run to find existing auth code before writing anything
2. Server-side session verification is present on every protected route (not client-side claims only)
3. Session cookies are configured with `httpOnly: true`, `secure: true` (prod), and `sameSite: 'lax'` minimum
4. Sign-out invalidates the session server-side — not just clears the cookie
5. `BETTER_AUTH_SECRET` (or equivalent secret) is read from environment — not hardcoded, not committed
6. Rate limiting is present on sign-in and sign-up endpoints
7. For multi-tenant features: every DB query is scoped to the authenticated user's org/tenant
8. End-to-end flow was verified: sign-up → sign-in → protected route access → sign-out → confirm redirect
9. No migration between auth providers was performed — existing provider was extended only
10. `.claude/progress.md` was updated on completion
