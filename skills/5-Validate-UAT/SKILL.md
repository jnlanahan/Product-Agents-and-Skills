---
name: 5-Validate-UAT
when_to_use: "User says 'user acceptance test', 'UAT', 'acceptance testing', 'test before launch', 'test with real users', or types /5-Validate-UAT."
description: MUST BE USED before migrating users to a new version or shipping a feature that changes existing behavior. Generates a structured user acceptance testing (UAT) checklist from the codebase and recent changes, walks through each scenario with the user, and records pass/fail/blocked results with a final ship/no-ship decision. Trigger on `/5-Validate-UAT`, "user acceptance test", "UAT", "acceptance testing", "test before launch", "test with real users".
---

# /5-Validate-UAT

You run a structured user acceptance test session. You generate the test plan from the codebase and recent changes, walk through each scenario with the user, and produce a UAT report with a clear pass/fail/conditional decision.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Generate and show the full test plan to the user before walking through any scenario — get agreement on scope before testing begins.
- Walk through every scenario even if early ones pass; partial UAT creates false confidence.
- A "Conditional" result requires the blocking issue to be fixed and re-tested before ship; do not treat it as a soft pass.

## When to Use

- Before migrating early users to a new version (W4: migrate-to-production)
- Before staged rollout of a high-impact feature (W3: add-feature)
- After `/5-Validate-Production-Readiness` finds behavioral issues that need human verification
- When automated tests pass but you need a human to confirm the actual UX

## Procedure

### Step 1: Understand what changed

Read:
- `git log --oneline -15` — recent commits
- `.claude/2-Define-Plan.md` if it exists — planned scope
- `CLAUDE.md` for project context

Run `stack-detector` to understand the tech so test steps are realistic (right URLs, right tooling).

### Step 2: Generate UAT checklist

Produce a checklist organized as: happy path → edge cases → error flows. Group by feature area.

Format each item as:
```
- [ ] **Scenario**: <what to do> | **Expected**: <what should happen>
```

Show the checklist. Ask:
> Are any scenarios missing? Any critical paths I haven't covered?

### Step 3: Walk through scenarios one at a time

For each scenario:
> **Testing**: [scenario name]
> **Steps**: [numbered steps for the user to follow in their browser/app]
> **Expected result**: [what they should see]
>
> → Pass / Fail / Blocked?

Record the result. If Fail or Blocked: ask what happened and log it verbatim.

### Step 4: Produce the UAT report

```markdown
## UAT Report — <feature/version>
Date: <YYYY-MM-DD>

### Summary
- Passed: N / N
- Failed: N
- Blocked: N

### Failed scenarios
- [scenario] — what happened

### Blocked scenarios
- [scenario] — blocker description

### Decision
[ ] PASS — ready for rollout
[ ] FAIL — blocking issues must be fixed before rollout
[ ] CONDITIONAL PASS — minor issues noted; rollout OK if <condition>
```

Ask the user to confirm the decision.

### Step 5: Save and act

Save report to `.claude/5-Validate-UAT-<feature>-<date>.md`.

If FAIL: offer to open triage sessions for failed scenarios.
> Want me to run `/5-Validate-Triage` for the failed scenarios? I'll work through each one.

---

## Common Scenarios by Feature Type

### Authentication
- Sign up with new email → confirm confirmation email sent (if applicable) → access granted
- Sign in with correct credentials
- Sign in with wrong password → correct error shown, account not locked
- Forgot password → reset email arrives → reset link works → new password accepted
- Access protected page while logged out → redirect to sign-in
- Sign out → protected page becomes inaccessible

### Payments (Stripe test mode)
- Checkout with test card `4242 4242 4242 4242` → payment succeeds → subscription activated
- Checkout with declined card `4000 0000 0000 0002` → clear error shown
- Webhook fires → subscription status updated in DB
- Customer Portal opens → cancel subscription → status updates

### File uploads
- Upload valid file → appears in UI with correct name/type
- Upload file exceeding size limit → validation error (not a crash)
- Upload disallowed file type → validation error
- Delete file → no longer accessible; storage cleaned up

### AI features
- Submit a prompt → response appears
- Submit a long prompt → no timeout or crash
- Submit empty input → validation error shown (not a crash)
- Check LangSmith dashboard → trace appears for each call

### Email
- Trigger sign-up → welcome email arrives within 60 seconds
- Trigger password reset → reset email arrives with working link
- Check Resend dashboard → delivery status shows "Delivered"

---

## Rules

- **Human-in-the-loop only** — this skill verifies behavior a human must observe. Do not automate the user interactions.
- **Test the unhappy path** — edge cases and errors catch more bugs than the happy path.
- **Blocked ≠ Failed** — a blocked scenario (can't even start it) needs different remediation than a failed one.
- **Produce a binary decision** — PASS / FAIL / CONDITIONAL PASS. "It mostly works" is not a UAT result.
- **Save the report** — it's the record that justifies the rollout decision.

## If Something Goes Wrong

- **Test scenario cannot be reproduced** — check that the test environment has the same data setup as described in the scenario; seed required data before retesting.
- **Stakeholder is unavailable to complete UAT** — document which scenarios are blocked and who is needed; do not issue a ship decision without covering all Critical and High scenarios.
- **All scenarios are blocked by the same root bug** — stop UAT, file a triage ticket for the root bug, and re-run UAT after the fix; do not issue a Conditional pass for a systemic blocker.