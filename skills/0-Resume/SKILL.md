---
name: resume
description: Run at the start of any new chat on a project with prior history. Reads .claude/progress.md and calls project-state-detector to give a 5–10 line orientation: current mode, last activity, open threads, recommended next step. Read-only.
---

# /resume

Recover session state when opening a project that has prior history. Run this at the top of any new conversation so you don't start cold. Takes under 30 seconds.

## Pre-flight

- Check for `.claude/progress.md`. If it does not exist, tell the user: "No progress log found — this looks like a fresh project. Run `/start` to initialize it."
- Check for `.claude/context.md`. Read it if it exists.

## Procedure

### Step 1: Read recent progress

Read `.claude/progress.md`. Extract the last 10 entries (or all entries if fewer than 10 exist). Note:
- Most recent skill invoked and its timestamp
- Any open threads or deferred decisions mentioned
- Output artifacts produced (what `.claude/` files exist)

### Step 2: Call project-state-detector

Run `project-state-detector` to get the current MODE, MATURITY, and RECOMMENDED_NEXT.

### Step 3: Output orientation summary

Produce a 5–10 line summary covering:

1. **Mode**: current PDLC phase and maturity (from detector)
2. **Last activity**: what was done and when (from progress.md)
3. **Artifacts on hand**: which `.claude/` files exist and are fresh
4. **Open threads**: anything explicitly marked deferred or unresolved in progress.md
5. **Recommended next**: the single most logical next step with one-line rationale

Format as plain prose or a short bulleted list — whichever is cleaner for the amount of content. Must fit on one screen.

### Step 4: Check for stale artifacts

If any artifact in `.claude/` has not been referenced in `progress.md` for 14+ days while the project shows recent activity, flag it: "⚠ `.claude/<file>` may be stale — last referenced <date>."

## Post-flight

This skill is read-only. Do not append to `progress.md` (that would pollute the log with meta-entries). If the user confirms they are resuming active work, the next skill they invoke will log the session normally.

## Constraints

- Never modify any files.
- If `.claude/` is empty or missing, redirect to `/start` immediately.
- Keep the output tight — the goal is orientation in under 30 seconds, not a full audit.
