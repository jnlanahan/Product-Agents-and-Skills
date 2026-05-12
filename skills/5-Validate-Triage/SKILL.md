---
name: triage
description: MUST BE USED when the user reports a bug or wants to investigate a problem. Listens conversationally, explores the codebase to find root cause, forms a hypothesis with file:line evidence, proposes 2+ fixes (root cause + workaround), and writes a complete bug report to `.claude/bugs/<short-name>.md`. Replaces standalone bug-investigation, bug-report, and QA-session skills. Trigger on `/triage`, "this is broken", "X doesn't work", "investigate this bug".
---

# /triage

You triage a bug end-to-end: listen, investigate, find root cause, propose fixes, write a complete bug report. The output is a markdown file at `.claude/bugs/<short-name>.md` that anyone (you, a teammate, future you) can pick up.

This skill replaces three older ones — bug-investigation, bug-report-doc, and QA-session — into a single conversational flow.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Reproduce the bug before investigating cause — if you can't reproduce it, say so and ask for steps to reproduce before proceeding.
- Form a root cause hypothesis with file:line evidence before proposing any fix; do not guess and patch.
- Write the bug report to `.claude/bugs/<short-name>.md` even for "obvious" bugs — the record helps future debugging.

## When to Use

- User reports a specific bug
- User wants a "QA session" — multiple bugs in one sitting
- User wants a written-up ticket for someone else to fix
- User is investigating intermittent or unclear problems

## When NOT to Use

- The fix is obviously a one-liner → just fix it
- It's a feature request, not a bug → use `/prd`
- It's an architectural concern, not a defect → use `/refactor`

## Procedure

For each bug the user reports:

### Step 1: Listen and lightly clarify

Let the user describe the problem in their own words. Ask **at most 2-3 short clarifying questions**:

- What did you expect vs what happened?
- Steps to reproduce (if not obvious)?
- Consistent or intermittent?

If they don't have logs or repro steps, walk them through capturing them (see Appendix). Otherwise move on.

### Step 2: Investigate

Run in parallel:
- `stack-detector` — to know what code we're looking at
- `pattern-finder` — only if you need to understand a specific area's conventions

Then dig:
- Read the user's reported error message verbatim
- Grep for the error message string OR a unique stack-trace identifier
- Read the file(s) most likely involved
- Check `git log -p <file>` for recent changes — bugs often correlate
- Look at adjacent code for similar patterns that work correctly
- Check existing tests (what's tested vs what's missing)

Do NOT propose changes here. You're investigating.

### Step 3: Form a hypothesis

What's the most likely root cause? Be specific. **Bad hypothesis**: "the API might be broken." **Good hypothesis**: "the price ID is passed as a string but Stripe expects a Price object reference, and `server/routes/checkout.ts:42` constructs it incorrectly when `STRIPE_PRICE_PRO` is undefined."

If uncertain, write multiple competing hypotheses ordered by likelihood. Don't fake confidence.

### Step 4: Design fixes

Propose at least two:

- **Fix A — Root cause** (recommended): addresses the underlying issue
- **Fix B — Quick workaround**: gets users unblocked while A is in review

If they're the same, say so explicitly. Otherwise the contrast helps the next engineer make a tradeoff.

For each fix, write a TDD plan: a numbered list of RED-GREEN cycles that capture the broken behavior and the minimal change to make it pass. Tests should verify behavior through public interfaces, not internals — the test should survive a future refactor.

### Step 5: Decide single bug or breakdown

Before writing the report, ask: is this **one bug** or **multiple distinct issues**?

**Break into multiple reports** when:
- The fix spans multiple independent areas
- Different people could work on each in parallel
- There are clearly separable failure modes

**Keep as one report** when:
- It's one behavior wrong in one place
- Multiple symptoms share one root cause

If breaking into multiple, create one file per issue, with cross-references in a `## Related` section.

### Step 6: Write the report

Write to `.claude/bugs/<kebab-case-short-name>.md`. Create the `.claude/bugs/` directory if missing. Use the template below. Print the file path and a one-line summary in chat.

### Step 7: Continue the session

If the user has more bugs, loop back to Step 1. Otherwise, summarize all bugs filed.

## Bug Report Template

```markdown
# Bug: <one-line summary>

*Status: Open · Severity: <Blocking everyone | Blocking some | Annoying | Cosmetic> · Filed: <YYYY-MM-DD>*

**Affected**: <users / pages / browsers / OS>
**First seen**: <date or "always" or "since deploy of <commit>">
**Reporter**: <user>

## What Happened

What the user actually experienced, in plain language. Use domain terminology from the project's glossary if one exists.

## What Should Happen

The expected behavior.

## Steps to Reproduce

1. <concrete step a developer can follow>
2. <step>
3. <step>
4. **Expected**: <what should happen>
5. **Actual**: <what happens>

## Logs

(verbatim — paste error messages and stack traces; sanitize secrets)

```
<paste here>
```

## Investigation

- <file:line> — <what I found>
- <file:line> — <what I found>
- Recent commits potentially related: <sha — message>
- Existing tests in this area: <what's covered, what's missing>

## Hypothesis

<the leading hypothesis, specific, with file:line citations>

If uncertain, list competing hypotheses ordered by likelihood.

## Proposed Fixes

### Fix A — Root cause (recommended)

<file:line>: <what to change>
**Why**: <brief>
**Risk**: <low/med/high — what could break>

**TDD plan**:
1. **RED**: write a test that <captures the broken behavior> — should fail because <reason>
   **GREEN**: <minimal change>
2. **RED**: write a test that <covers an adjacent edge case>
   **GREEN**: <minimal change>

### Fix B — Quick workaround

<file:line>: <what to change>
**Why**: <brief>
**Risk**: <low/med/high>

## Acceptance Criteria

- [ ] Original repro no longer reproduces
- [ ] New tests pass
- [ ] Existing tests still pass
- [ ] No regression in <related feature>

## Out-of-Scope Notes

Things noticed during investigation that aren't this bug.
```

## Rules

- **Listen first; investigate second.** Don't start grepping before the user has explained the problem.
- **Verbatim quoting of errors.** Don't paraphrase — error messages are searchable terms.
- **File:line for every claim.** No vague "the auth code might be broken."
- **Hypothesis can be wrong.** Mark uncertainty: "leading hypothesis (60% confidence)" beats false confidence.
- **Two fixes minimum.** Forces tradeoff thinking. If they're truly the same, say so.
- **Don't fix it in this skill.** That's a separate explicit ask. The report is the artifact.
- **Sanitize logs.** Strip API keys, JWT tokens, password hashes, full URLs with secrets before writing into the report.
- **Tests describe behavior, not implementation.** A test that breaks when an internal function is renamed is the wrong test.
- **Save reports to `.claude/bugs/`** — one file per bug. The user can `git add` whichever they want to commit.
- **Recommend `git add .claude/bugs/<file>.md`** in the chat summary so the user remembers to commit it.

## Appendix — Capturing logs (if user doesn't have them)

### Frontend (browser)

1. Open DevTools (F12)
2. Console tab → click the clear icon
3. Reproduce the bug
4. Right-click in console → Save as → `console-<date>.txt`
5. Network tab → reproduce again → right-click → Save all as HAR with content → `network-<date>.har`

### Backend

```bash
# Terminal where the server runs:
npm run dev 2>&1 | tee server-logs.txt
# Reproduce the bug
# Ctrl+C when done
```

### Sentry

If Sentry is wired, the error is probably already there. Check sentry.io → your project → recent issues. Copy the issue URL into the report.

## If Something Goes Wrong

- **Cannot reproduce the bug** — ask for exact steps, browser/OS version, and whether it occurs in incognito mode; document the reproduction failure in the bug report rather than guessing at root cause.
- **Root cause is not found in the codebase** — check git log for recent changes to the affected area (`git log --oneline -20 -- path/to/file`); the bug may be a regression.
- **Multiple equally plausible root causes** — document all candidates in the bug report with evidence for each; let the user choose which to fix first.