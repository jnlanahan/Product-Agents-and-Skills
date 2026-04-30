---
name: 5-Validate-Production-Readiness
description: MUST BE USED before a production launch or after a big change to the critical path. Runs a deep production-readiness audit and returns a severity-graded report (Critical/High/Medium/Low) with file:line citations and a recommended fix order. Trigger on `/check-production`, "is this ready for production", "audit before launch", or any pre-deploy review request.
---

# /check-production

You orchestrate a deep production-readiness audit. The actual audit work is delegated to the `prod-readiness-auditor` agent; this skill assembles the inputs, runs supporting scans in parallel, and presents the result with a clear "what to fix in what order" recommendation.

## When to Use

- Before going live with a new app
- After a major refactor or new integration
- Periodic check (quarterly) on a production app
- After running `/next-steps` and the user wants to drill down on Stage-4 hardening

## Procedure

### Step 1: Set context

Run in parallel:
- `stack-detector` — required input for the auditor
- `codebase-classifier` — calibrates audit severity (greenfield gets gentler grading; vibe-coded gets stricter)
- `secret-scanner` — secrets findings stand alone; very high signal
- `dependency-currency-checker` — outdated deps factor into the report

These are independent — call all four in one message.

Read `_stack-preferences.md` and `_adaptation-playbook.md`.

### Step 2: Run the auditor

Call `prod-readiness-auditor` with full context:

> Run a production-readiness audit on the project at the current working directory.
>
> Stack profile:
> <paste stack-detector output>
>
> Classification:
> <paste codebase-classifier output>
>
> Pre-scanned secrets: <count from secret-scanner; brief summary>
> Outdated dependencies: <list from currency-checker>
>
> Cover all 8 areas. Cite file:line for every finding. Be honest about severity.

### Step 3: Assemble the final report

Take the auditor's output. Add:

- **Cross-cutting actions**: things that aren't a single file finding but require multiple changes (e.g., "add CI" affects deploy, tests, package.json scripts)
- **Recommended fix order**: not just severity. Ordering also considers dependencies (you can't add Sentry source maps without Sentry; you can't add CI test gates without tests).
- **Estimated effort**: rough hours per Critical/High item. Helps the user plan.

### Step 4: Present and offer follow-up

Show the report. Ask:

> Want me to fix anything from this report now? Tell me which findings (e.g., "fix all Critical and the rate-limiting one") and I'll work through them, getting confirmation before each substantive change.

If yes: walk through fixes one at a time, getting approval before each. Do NOT batch fixes — each is its own commit.

If no: save the report to `PRODUCTION_AUDIT_<date>.md` in the project root and end the skill.

## Report Format

The auditor returns its standard format. You wrap it like this:

```markdown
# Production Readiness Audit
**Project**: <name>
**Date**: <YYYY-MM-DD>
**Stack**: <one-line summary>
**Classification**: <greenfield | wired | vibe-coded>

## Executive Summary
<2-3 sentences: how production-ready is this, what's the biggest risk, what's the recommended next step>

## Severity Counts
- Critical: <N>
- High: <N>
- Medium: <N>
- Low: <N>

## Critical Findings (fix before launch — show-stoppers)
<from auditor — verbatim>

## High Findings (fix in first month of production)
<from auditor>

## Medium Findings (fix when you have time)
<from auditor>

## Low Findings (nice-to-haves)
<from auditor>

## Cross-Cutting Actions
<things that span multiple files>

## Recommended Fix Order
1. <action> — <one-line rationale> (~<N> hours)
2. <action> — <rationale> (~<N> hours)
3. <action> — <rationale> (~<N> hours)

## Positive Observations
<from auditor — keep morale up>

## Next Steps
<which skills to run after this; e.g., "/add-monitoring then /deploy">
```

## Common Patterns by Classification

### Greenfield project
Audit will mostly find Mediums (no tests, no CI, missing nice-to-haves). That's expected. Don't overstate severity. Recommend running `/setup-project` if not done.

### Wired project (this template)
Audit should find few Criticals. Most findings will be specific things to harden (e.g., "rate limit endpoint X is unprotected"). Treat as a polish exercise.

### Vibe-coded project
Audit will surface real risks — committed secrets, missing webhook verification, no CI, no error tracking. Lead with rotation if secrets are exposed. The recommended fix order matters most here; sequencing prevents the user from doing things in dangerous orders (e.g., enabling Sentry source maps before rotating Sentry auth token in CI).

## Rules

- **Delegate to the agent.** Don't try to audit in this skill's context — that's what `prod-readiness-auditor` is for.
- **Parallel scans.** stack-detector, classifier, secret-scanner, currency-checker all run in parallel before the auditor.
- **Save the report.** It's a document the user will reference. Write it to disk if they decline immediate fixes.
- **No silent fixes.** If the user opts in to fixes, get confirmation per finding. Auditing is read-only; fixing is collaborative.
- **Don't bundle fixes by file.** Each finding is its own commit, even if two findings live in the same file. Easier to review and revert.
