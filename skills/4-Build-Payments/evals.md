# Evals: /add-payment

Binary pass/fail criteria. Grading agent: check output against each criterion and return PASS or FAIL.
For each FAIL provide one line of reason. Do not add criteria beyond what is listed.

1. Webhook signature verification using `stripe.webhooks.constructEvent` is present before any processing
2. `STRIPE_WEBHOOK_SECRET` is read from environment — not hardcoded, not committed to git
3. Idempotency handling is in place so duplicate webhook events do not double-process
4. No raw card data is handled in application code — collection goes through Stripe Elements or Checkout
5. Test-mode keys (`sk_test_...`) are used during development — no live keys wired prematurely
6. `stripe listen` verification was performed (or documented as a required step) before committing
7. Customer Portal is used for subscription management — no custom cancel/upgrade UI built
8. At least one failed-payment state is handled (e.g., `invoice.payment_failed`)
9. DB update was verified after a successful test webhook — not assumed correct
10. `.claude/progress.md` was updated on completion
