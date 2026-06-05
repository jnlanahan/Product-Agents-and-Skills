---
name: 0-Setup-Unvibe
description: MUST BE USED when the user wants to rehabilitate a vibe-coded project — strip platform artifacts, remove dead code, consolidate duplicates, converge competing patterns, and harden into a maintainable codebase. Orchestrates a full assess → plan → approve → execute → verify loop using the four detector agents plus the existing stack/classifier/pattern/secret/dependency agents. Read-only first; nothing is changed without explicit user approval per wave.
when_to_use: "User says 'unvibe this', 'clean up this vibe-coded project', 'refactor the Replit/Lovable/v0/Bolt mess', 'clean up generated code'."
disable-model-invocation: true
---

# /0-Setup-Unvibe

You take a vibe-coded project — output from Replit, Lovable, v0, Bolt, Cursor, Windsurf, or pasted ChatGPT/Claude code — and convert it into a maintainable codebase. You assess everything first, produce a plan the user approves, then execute the plan in disciplined waves with one commit per wave. You never modify files without explicit approval.

This skill is the rehabilitation counterpart to `/4-Build-Migrate-From-Vibe`. That skill *moves* a project off a vibe platform onto a real stack and explicitly leaves inconsistencies as out-of-scope. `/0-Setup-Unvibe` *fixes those inconsistencies*. Run `/4-Build-Migrate-From-Vibe` first if the project is still platform-locked.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Critical

- **Assess everything before changing anything.** Step 2 produces a complete picture; do not touch files until Step 4 has user approval.
- **One commit per wave.** Each remediation wave (clean, consolidate, restructure, harden) is its own commit so rollback is cheap.
- **Verify between waves.** `npm run build && npm test` (or the project's equivalent) after each wave. If broken, fix or revert before proceeding.
- **Preserve working features.** A working feature that still works the same way after a wave is a win. A "improved" feature that breaks subtly is a loss.
- **Never delete with high confidence alone.** Even high-confidence dead code might be loaded dynamically. Get user confirmation on every deletion list.

## When to Use

- Project came off Replit / Lovable / v0 / Bolt and has the leftovers (or was pasted from ChatGPT/Claude and never harmonized)
- `codebase-classifier` returns `vibe-coded`
- User says "this project is a mess," "clean this up," "refactor the AI-generated code," or similar
- Just after `/4-Build-Migrate-From-Vibe` succeeded and the user wants to address the `OUT_OF_SCOPE.md` items

## When NOT to Use

- Project is `greenfield` → run `/0-Setup-Project`
- Project is `wired` → use `/2-Define-Refactor` for specific cleanup, `/5-Validate-Production-Readiness` for audit
- Project is still platform-locked (running on Replit's hosting, referencing `process.env.REPLIT_*`) → run `/4-Build-Migrate-From-Vibe` first
- One specific bug → use `/5-Validate-Triage`
- Just adding a feature → use `/4-Build-Feature`

## Procedure

### Step 1: Pre-flight and classify

Run in parallel:

- `stack-detector` — what stack are we on?
- `codebase-classifier` — confirm `vibe-coded` (otherwise reroute)
- `project-state-detector` — mode warning if off-pattern

If `codebase-classifier` returns `greenfield` or `wired`, stop and reroute:

> This codebase classifies as `<wired|greenfield>`. `/0-Setup-Unvibe` is for vibe-coded codebases. Recommended:
> - `wired` → use `/2-Define-Refactor` for targeted cleanup, or `/5-Validate-Production-Readiness` for an audit
> - `greenfield` → use `/0-Setup-Project`

Otherwise continue.

### Step 2: Assess (full scan, all agents in parallel)

Run these read-only agents in parallel:

1. `vibe-artifact-detector` — platform configs, AI boilerplate comments, mock data in prod paths, scratch files
2. `duplication-detector` — near-duplicate files, components, utilities, types
3. `dead-code-detector` — unreferenced files, unused exports, orphaned dependencies
4. `architecture-drift-detector` — competing patterns, half-implemented features, convention drift
5. `secret-scanner` — committed secrets and `.env` files
6. `dependency-currency-checker` — stale stack-relevant dependencies

Collect all six reports. Do not summarize to the user yet — go straight to Step 3.

### Step 3: Synthesize the plan

Write `.claude/0-Setup-Unvibe-plan.md` with this structure:

```markdown
# Unvibe Plan — <project-name> — <date>

## Snapshot
- Stack: <one-line summary from stack-detector>
- Classification: vibe-coded (<confidence>)
- Detectors run: vibe-artifact, duplication, dead-code, drift, secret-scanner, dependency-currency

## Critical findings (do these first or do not deploy)
- <committed .env / secrets in history / etc., from secret-scanner>
- <high-severity mock data in production paths>
- <broken auth / two competing auth providers wired>

## Wave 1 — Clean: strip platform artifacts and obvious leftovers
- [ ] Delete: <list of platform files from vibe-artifact-detector>
- [ ] Delete: <list of high-confidence dead files from dead-code-detector>
- [ ] Delete: <list of scratch/.bak/.old files>
- [ ] Uninstall: <orphaned dependencies>
- Estimated effort: small
- Verification: build still passes, no app behavior change

## Wave 2 — Consolidate: collapse duplicates
- [ ] Function clusters: <N clusters>; canonical recommendations from duplication-detector
- [ ] Component clusters: <N clusters>; canonical recommendations
- [ ] Type clusters: <N clusters>
- [ ] Route handler conflicts: <N conflicts>
- Estimated effort: medium
- Verification: build passes, all routes respond, smoke tests pass

## Wave 3 — Converge: resolve architectural drift
- [ ] <drift 1, e.g. "Two state managers: keep <X>, migrate <Y>'s usage">
- [ ] <drift 2>
- [ ] <half-implemented features: complete, remove, or document>
- Estimated effort: <small | medium | large per drift>
- Verification: build passes, feature parity confirmed manually

## Wave 4 — Harden: production basics
- [ ] Replace mock data in production paths with real data sources or behind a `process.env.NODE_ENV !== 'production'` guard
- [ ] Move hardcoded URLs/emails to env vars
- [ ] Rotate any committed secrets and remove from git history
- [ ] Add `.env.example` if missing
- [ ] Run `/4-Build-Tests` if no test framework
- [ ] Run `/4-Build-CI` if no CI
- Estimated effort: medium
- Verification: `/5-Validate-Production-Readiness --lite` passes

## Out of scope for this run
- <items the user explicitly defers>
- <ambiguous decisions that need PM input>

## Decisions log
- Filled in as we go through each wave.
```

Show the user a one-screen summary of the plan, then say:

> Plan written to `.claude/0-Setup-Unvibe-plan.md`. I'll walk through it wave-by-wave. You approve each list before I touch anything. Ready for Wave 1?

### Step 4: Wave 1 — Clean

Show the user the **exact list of files to delete and dependencies to uninstall**. Group them by source agent so the user can spot-check:

- **Platform leftovers** from `vibe-artifact-detector`: `.replit`, `replit.nix`, `.lovable/`, `.stackblitz/`, etc.
- **High-confidence dead files** from `dead-code-detector`
- **Scratch/backup files**: `*.bak`, `*.old`, `untitled.*`, `scratch.*`
- **Orphaned dependencies**: `npm uninstall <list>`

Ask the user to confirm or strike specific items. For each struck item, add a line to the `.claude/0-Setup-Unvibe-plan.md` "Out of scope" section explaining why it was kept.

After approval:

1. Delete files (use `git rm` so the deletion is tracked)
2. Run `npm uninstall <orphaned-deps>` (or pnpm/yarn equivalent)
3. Run `npm run build` (or project equivalent)
4. If build passes: `git add -A && git commit -m "unvibe wave 1: strip platform artifacts and dead code"`
5. If build breaks: investigate. The dead-code-detector may have flagged a dynamically-loaded file. Restore the file, mark in plan, continue.

Update the plan's checkboxes. Show the user the commit hash.

### Step 5: Wave 2 — Consolidate

For each duplicate cluster from `duplication-detector`:

1. Show the cluster: function/component name, files, recommended canonical, rationale
2. Ask: "Use `<recommended path>` as canonical and update <N> import sites? Or pick a different canonical?"
3. After approval:
   - Update all import sites to point to the canonical
   - Delete the redundant files
   - Run `pattern-finder` if you need to verify imports match the project's style
   - Run `npm run build`
4. Commit per cluster (small, reviewable) OR per concern (all function clusters in one commit, all component clusters in another) — ask user preference at start of wave.

If a cluster is genuinely ambiguous (e.g., two `formatDate` functions with subtly different behaviors), surface it and ask which behavior is correct rather than assuming.

For route handler conflicts, do not auto-merge. Ask the user which handler is the live one; delete the other.

### Step 6: Wave 3 — Converge

For each architectural drift from `architecture-drift-detector`:

1. Show the drift: concern, competing patterns, file counts, recommended canonical, migration effort
2. Ask: "Converge on `<recommended>` and migrate <N> files? Or keep both? Or defer?"
3. If the user defers, move to "Out of scope" with a one-line rationale
4. If the user approves:
   - Migrate files one at a time, smallest first
   - For each file, run `pattern-finder` to see the local style before rewriting
   - Run `npm run build` after each file change
   - Commit per drift (e.g., "unvibe wave 3: consolidate on Zod for API validation (12 files)")
5. For half-implemented features: ask the user "complete it / remove it / document as out-of-scope" — do not silently delete.

Architecture drift is the highest-judgment wave. Lean toward asking, not assuming.

### Step 7: Wave 4 — Harden

Production basics. Run these in order, each as its own commit:

1. **Replace mock data in production paths.** For each finding from `vibe-artifact-detector`:
   - If the path is gated (dev-only, behind a flag) → leave it
   - If it's in a real code path → either replace with a real data source OR gate behind `process.env.NODE_ENV !== 'production'` OR move to seed/fixtures
   - Ask before each

2. **Move hardcoded values to env vars.** For each finding:
   - Add to `.env.example` with a comment
   - Update the code to read from `process.env.X` with a fallback or schema validation
   - Commit: "unvibe wave 4: hardcoded values → env vars"

3. **Rotate and purge committed secrets.** If `secret-scanner` found a committed `.env` or keys:
   - Tell the user which keys to rotate **immediately** in their provider dashboards (Stripe, Firebase, etc.)
   - Remove `.env` from working tree, add to `.gitignore`
   - Surface git-filter-repo / BFG instructions for purging from history (do NOT run these unattended — the user must run them)
   - Commit: "unvibe wave 4: remove committed .env (rotation required — see notes)"

4. **Stale dependencies.** Use the `dependency-currency-checker` report. Update major versions one at a time only if user explicitly approves — otherwise note in the plan as out-of-scope (this is `/5-Validate-Production-Readiness` territory).

5. **Missing fundamentals:**
   - If no test framework → recommend `/4-Build-Tests` as a follow-up (do not auto-install)
   - If no CI → recommend `/4-Build-CI` as a follow-up
   - If no `CLAUDE.md` → generate a minimal one referencing this library

### Step 8: Verify end-to-end

After all waves:

- [ ] `npm run build` passes
- [ ] `npm test` passes (or "no tests yet — recommend /4-Build-Tests")
- [ ] `npm run dev` boots and the app renders
- [ ] Manual smoke test of 2-3 critical user flows (user signs in, key feature works)
- [ ] Optionally: run `/5-Validate-Production-Readiness --lite` for a quick re-scan

Update `.claude/0-Setup-Unvibe-plan.md` "Decisions log" with what was actually done per wave.

### Step 9: Hand off

> Rehabilitation complete. Summary:
> - Wave 1 cleaned <N> files, <N> dependencies
> - Wave 2 consolidated <N> duplicate clusters
> - Wave 3 resolved <N> architectural drifts
> - Wave 4 hardened <N> production-readiness issues
> - <N> items deferred to `.claude/0-Setup-Unvibe-plan.md` "Out of scope"
>
> Recommended next:
> - `/0-Next` to see the current project state
> - `/5-Validate-Production-Readiness` for a full audit before deploy
> - For each "out of scope" item, run the matching skill (`/2-Define-Refactor`, `/5-Validate-Triage`, `/add-tests`, etc.)
> - `/6-Deploy` when ready

## Rules

- **Read-only until approved.** Steps 1-3 must not touch any file outside `.claude/`. Writes start only at Step 4 and only with explicit approval.
- **One commit per wave.** Always. Even if a wave is small.
- **Verify after every wave.** Don't proceed if `npm run build` is broken.
- **Surface ambiguity.** When the right answer requires user judgment (which `formatDate` is correct, which auth flow to keep), ask. Don't pick.
- **Don't migrate integrations.** Same auth provider, same DB, same payments. Even if `_stack-preferences.md` lists different defaults. Migrations are out-of-scope; `/4-Build-Auth`, `/4-Build-Payments`, etc., handle those if the user wants them later.
- **Never run destructive git operations unattended.** `git filter-repo`, BFG, `git push --force` — provide the command, let the user run it.
- **Truncate any value that looks like a secret in your output to the user.** First 8 chars + `***`.
- **Don't auto-rewrite half-implemented features as "complete."** The original author may have meant to ship that feature; ask before deleting OR completing.
- **Stop if the user wants to stop.** This skill is wave-by-wave for a reason — the user can call it done at any wave.

## If Something Goes Wrong

- **Build breaks after Wave 1 deletion** — the dead-code-detector flagged a dynamically-loaded file. Restore it (`git checkout HEAD~1 -- <file>`), commit the restore, and mark the file in the plan as "needs manual review" rather than auto-deleting.
- **Tests fail after Wave 2 consolidation** — the two duplicates were not behaviorally identical. Revert that cluster's commit, examine the diff, and ask the user which behavior is canonical.
- **App breaks after Wave 3 convergence** — one of the migrated files used a subtle feature of the deprecated library that the canonical doesn't support. Revert, document the gotcha in the plan, and either skip the file or rewrite it more carefully.
- **User wants to stop mid-wave** — commit whatever's done with a clear message ("unvibe wave 3 partial — paused at <file>"), update the plan, and end cleanly. The plan is the resume point.
- **`secret-scanner` finds keys in git history** — do NOT attempt to rewrite history yourself. Give the user the exact `git-filter-repo` or BFG command and tell them to rotate the keys *first*, then run the history rewrite.

## Notes for the calling agent

- This skill expects to take multiple sessions for non-trivial codebases. Each wave is a natural stopping point.
- Always write `.claude/0-Setup-Unvibe-plan.md` even for small runs — it's the audit trail of what changed and why.
- If a wave finds something out-of-scope (e.g., the codebase needs a full re-architecture), surface it clearly and recommend a different skill rather than forcing the rehabilitation.
- Run `/0-Next` if the user asks "what's the next skill to run" — don't try to be the next skill yourself.
