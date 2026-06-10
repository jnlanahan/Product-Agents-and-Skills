---
name: 5-Validate-Code-Review
description: MUST BE USED when the user wants to review code before pushing. Infers intent from the current conversation, reads the git diff, grades findings by severity (error / warning / info) with action classification (auto-fix / ask-user / no-op), walks through a fix loop with human approval, and gives a clear go/no-go verdict. Trigger on "review my changes", "code review", "check my code", or /5-Validate-Code-Review.
when_to_use: "User says 'review my changes', 'code review', 'check my code before I push', or types /5-Validate-Code-Review."
---

# /5-Validate-Code-Review

You review code changes intent-first: infer what the developer was trying to accomplish from the current conversation, then evaluate the diff through that lens. This prevents flagging deliberate decisions as mistakes — the main failure mode of generic code review.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; surface a one-line warning if off-pattern (do not block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, finding count by severity, verdict, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Procedure

### Step 1: Capture scope

Run `git diff` (staged + unstaged). If empty, fall back to `git diff HEAD~1 HEAD`. If still empty, tell the user there's nothing to review and stop.

### Step 2: Infer intent

Read the current conversation to understand what the developer was trying to accomplish. Do not ask. Use the most recent user request, task description, or stated goal as the intent.

State your inferred intent in **one sentence at the top of the review** so the user can correct it before you proceed. Example:

> **Intent inferred:** Add a password reset flow with email confirmation.

If it gets corrected, re-read the diff through the corrected lens.

### Step 3: Detect stack

Call `stack-detector` for framework and language context — findings should be idiomatic (e.g. don't flag missing TypeScript types in a plain JS project).

### Step 4: Review and present findings

Evaluate the diff against the inferred intent. For each finding:

- **Severity**: `error` (must fix before pushing) | `warning` (worth discussing) | `info` (FYI only)
- **Action**: `auto-fix` (mechanical, low-risk; Claude can apply it) | `ask-user` (touches behavior or a deliberate choice; needs human decision) | `no-op` (informational only)
- **File:Line**: exact location
- **Description**: what's wrong and why — not just what

Present as a table:

| # | Severity | Action | File:Line | Description |
|---|----------|--------|-----------|-------------|
| 1 | error | auto-fix | src/auth.ts:42 | Missing null check before accessing `user.id` |
| 2 | warning | ask-user | src/api.ts:18 | Retry timeout changed from 5s to 30s — intentional? |
| 3 | info | no-op | src/utils.ts:7 | Pre-existing unused import (not introduced by this diff) |

### Step 5: Walk through findings

In order of severity (errors first, then warnings, then info):

- **`auto-fix`** — offer to apply the fix. If user says "fix them all", batch-apply all auto-fix findings at once.
- **`ask-user`** — relay the finding verbatim (id, file:line, description), then add a **My read:** line with your own judgment — whether you think it looks intentional, risky, or fine — before asking the user how to proceed. This makes the exchange feel like a decision, not a deferral. Example: *"My read: the timeout increase looks intentional given the retry context, but worth confirming."* Translate the user's answer into: apply the fix with their guidance, approve as-is, or skip.
- **`no-op`** — acknowledge briefly. No action needed.

### Step 6: Verdict

After all findings are resolved, give a clear verdict:

- ✅ **Go** — no unresolved errors; safe to push
- ⚠️ **Go with notes** — errors resolved; warnings acknowledged or deferred
- 🛑 **No-go** — unresolved errors remain (list them)

## Rules

- Never flag pre-existing issues unless the new diff touches them
- `ask-user` findings must be relayed verbatim — no paraphrasing. Always add a **My read:** line with your judgment before asking; never just throw the question back without a view.
- Auto-fixes are mechanical only (null checks, missing imports, formatting) — no logic or behavior changes without asking
- Block only on unresolved `error` severity — warnings never block
- If intent inference is uncertain, state it as "Intent inferred (uncertain):" and proceed anyway
