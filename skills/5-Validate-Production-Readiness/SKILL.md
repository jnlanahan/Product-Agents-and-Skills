---
name: check-production
when_to_use: "User says 'is this ready for production', 'audit before launch', 'pre-deploy check', 'production checklist', or types /check-production."
description: MUST BE USED before a production launch or after a big change to the critical path. Runs a deep production-readiness audit and returns a severity-graded report (Critical/High/Medium/Low) with file:line citations and a recommended fix order. Add `--lite` for a fast 30-second sanity check that skips the full auditor. Trigger on `/check-production`, "is this ready for production", "audit before launch", or any pre-deploy review request.
---

# /check-production

You orchestrate a deep production-readiness audit. The actual audit work is delegated to the `prod-readiness-auditor` agent; this skill assembles the inputs, runs supporting scans in parallel, and presents the result with a clear "what to fix in what order" recommendation.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- This skill is audit-only — it does not apply fixes. Resist the urge to fix findings mid-audit; finish the full report first.
- Do not substitute `--lite` mode for a full audit before a first production launch; use `--lite` only for repeat checks after fixes are applied.
- Critical-severity findings must be resolved before launch — do not ship with any open Critical items.

## When to Use

- Before going live with a new app
- After a major refactor or new integration
- Periodic check (quarterly) on a production app
- After running `/next-steps` and the user wants to drill down on Stage-4 hardening
- `--lite` flag: quick pre-deploy sanity check for hotfixes (W6) or personal tools (W8)

## `--lite` Mode (fast sanity check)

When invoked as `/check-production --lite`, skip the full auditor and run a quick pass instead:

1. Run `secret-scanner` only (fastest, highest-signal scan)
2. Run `stack-detector` to confirm the expected stack is in place
3. Check that `npm run build` (or equivalent) passes
4. Check `/api/health` returns 200
5. Report: pass (no critical secrets found, build green, health OK) or fail (list what broke)

Use `--lite` for: hotfix pre-deploy checks, personal tools (W8), or any situation where a 30-second check is better than a 10-minute one. **Not a substitute for the full audit before a first production launch.**

---

## Full Procedure

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

→ See [audit-output-format.md](references/audit-output-format.md) for the full report template structure and calibration notes by project classification (greenfield / wired / vibe-coded).

## Rules

- **Delegate to the agent.** Don't try to audit in this skill's context — that's what `prod-readiness-auditor` is for.
- **Parallel scans.** stack-detector, classifier, secret-scanner, currency-checker all run in parallel before the auditor.
- **Save the report.** It's a document the user will reference. Write it to disk if they decline immediate fixes.
- **No silent fixes.** If the user opts in to fixes, get confirmation per finding. Auditing is read-only; fixing is collaborative.
- **Don't bundle fixes by file.** Each finding is its own commit, even if two findings live in the same file. Easier to review and revert.

## If Something Goes Wrong

- **prod-readiness-auditor times out** — run `--lite` mode for a quick scan and follow up with a full audit on a targeted area; do not skip the audit entirely.
- **Critical finding is disputed** — document the disagreement in `.claude/next-steps.md` with the justification for overriding; do not silently mark it resolved without evidence.
- **Audit finds no Critical items but the app feels fragile** — run `secret-scanner` and `dependency-currency-checker` separately; the auditor may have incomplete coverage for the specific stack.