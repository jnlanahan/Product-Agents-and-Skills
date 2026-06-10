# Memory: /add-payment

Lessons from past runs. Max 7 entries. Newest first.
When adding entry 8, remove the oldest.
If a pattern appears twice, promote it to SKILL.md or evals.md, then delete it here.

- For Hono apps: webhook raw body = `c.req.text()`. Register webhook route without auth middleware — Stripe signature IS the auth.
- Dev/mock auth bypass (e.g., `/dev-login`) must be removed or disabled before testing payment flows or test-mode results are unreliable.
- Admin user who bypasses payment limits is a near-universal need — ask about it in Step 3 rather than waiting for the user to raise it.
