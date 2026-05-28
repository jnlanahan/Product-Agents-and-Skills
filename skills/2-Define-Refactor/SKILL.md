---
name: refactor
description: MUST BE USED when the user wants to refactor code — either to find opportunities ("where's our shallow code?") or to plan a known refactor with safe, tiny commits. Combines opportunity-scanning and plan-and-execute modes. Includes Claude-Code-specific refactoring best practices. Writes the plan to `.claude/refactor-plan.md`.
when_to_use: "User says 'refactor this', 'clean up the code', 'find shallow code', 'improve code quality', 'restructure this'."
---

# /refactor

Two entry modes — pick based on user intent:

- **Find mode**: scan the codebase for deepening opportunities (turn shallow modules into deep ones, improve testability, improve AI-navigability). Use when the user asks "where should I refactor?" or "what can be improved?"
- **Plan mode**: the user already has a refactor in mind. Interview, hammer scope, break into tiny commits. Use when the user asks "help me plan this refactor" or names a specific area.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Before You Start

- For Plan mode: scope the refactor target before starting — open-ended refactors balloon quickly.
- Run on a clean working tree; commit or stash existing changes first so the refactor diff is unambiguous.

Output: `.claude/refactor-plan.md` (the chosen candidate's full plan). Recommend the user commit it.

## Glossary (use these terms exactly)

Consistent language is the point — don't drift into "component," "service," "API," or "boundary." Full definitions in [LANGUAGE.md](LANGUAGE.md).

- **Module** — anything with an interface and an implementation
- **Interface** — everything a caller must know to use the module: types, invariants, error modes, ordering, config
- **Implementation** — the code inside
- **Depth** — leverage at the interface; deep = small interface hiding meaningful complexity
- **Seam** — where an interface lives; a place behavior can change without editing in place
- **Adapter** — a concrete thing satisfying an interface at a seam
- **Leverage** — what callers get from depth
- **Locality** — what maintainers get from depth: change, bugs, knowledge concentrated in one place

Key principles:

- **Deletion test**: imagine deleting the module. Does complexity vanish (it was a pass-through), or reappear across N callers (earning its keep)?
- **The interface is the test surface.**
- **One adapter = hypothetical seam. Two adapters = real seam.**

---

## Find Mode

### 1. Explore

Read existing documentation first:

- `.claude/glossary.md` (or any `UBIQUITOUS_LANGUAGE.md` / `CONTEXT.md` in older repos) — domain terms
- Any `docs/adr/` directory — decisions you should not re-litigate

If those don't exist, proceed silently. Don't flag absence.

Then explore the codebase organically (use the `Explore` subagent for breadth). Don't follow rigid heuristics — note where you experience friction:

- Where does understanding one concept require bouncing between many small modules?
- Where are modules **shallow** — interface nearly as complex as implementation?
- Where have pure functions been extracted just for testability, but the real bugs hide in how they're called?
- Where do tightly-coupled modules leak across their seams?
- Which parts are untested or hard to test through the current interface?

Apply the **deletion test** to anything you suspect is shallow.

### 2. Present candidates

Numbered list. For each:

- **Files** — modules involved
- **Problem** — current friction
- **Solution** — plain English description of the change
- **Benefits** — explained in terms of locality, leverage, and how tests would improve

Use glossary vocabulary for the domain ("the Order intake module" — not "FooBarHandler"). Use [LANGUAGE.md](LANGUAGE.md) vocabulary for architecture.

**ADR conflicts**: if a candidate contradicts an existing ADR, surface only when friction is real enough to warrant revisiting. Don't list every theoretical refactor an ADR forbids.

Do NOT propose interfaces yet. Ask: "Which would you like to plan?"

### 3. Drop into Plan Mode for the chosen candidate

---

## Plan Mode

### 1. Gather problem description

Ask the user for a long, detailed description of the problem they want to solve and any potential ideas for solutions.

### 2. Explore the repo

Verify the user's assertions and understand current state. Read the relevant files. Look at sibling code for conventions.

### 3. Consider alternatives

Ask whether they've considered other options. Present alternatives. See [INTERFACE-DESIGN.md](INTERFACE-DESIGN.md) for how to spawn parallel sub-agents to explore multiple deepened-interface designs.

### 4. Interview deeply

Walk the design tree with them — constraints, dependencies, the shape of the deepened module, what sits behind the seam, what tests survive.

Side effects happen inline:

- **Naming a deepened module after a concept not in the glossary?** Add the term (call `/glossary` or update inline).
- **User rejects a candidate with a load-bearing reason?** Offer an ADR — only when the reason would actually be needed by a future explorer to avoid re-suggesting the same thing.

### 5. Hammer the scope

Be explicit about what's in and out. The most common failure mode is scope creep dragging a 2-day refactor into a 2-week saga.

### 6. Check test coverage

Look at the area's existing tests. If coverage is thin, ask: characterization tests first to lock in current behavior? Or do we accept the risk and move?

### 7. Break into tiny commits

Martin Fowler: "make each refactoring step as small as possible, so that you can always see the program working." Each commit must:

- Leave the codebase in a working, tested state
- Change only one thing
- Be revertible without unwinding subsequent commits

### 8. Write `.claude/refactor-plan.md`

## Refactor Plan Template

```markdown
# Refactor Plan: <name>

*Status: Ready · Last updated: <YYYY-MM-DD>*

## Problem

The problem from the developer's perspective. What is the current pain?

## Solution

The intended end state. What does the deepened module look like? What does the seam look like?

## Decision Document

Record decisions that would not be obvious from the diff:

- Modules built/modified and their new interfaces
- Architectural decisions and their reasoning
- Schema changes and their reasoning
- API contracts (if any change publicly)
- Specific behavioral interactions

Do NOT include file paths or code snippets — they go stale fast.

## Commits

A long, detailed implementation plan. Each item is one tiny commit. Each commit leaves the codebase working.

1. **<imperative summary>** — <what changes; what stays>; tests: <run; expect green>
2. **<imperative summary>** — <what changes>; tests: <run>
3. ...

The last commit should be the one that finally deletes the old code (if applicable). Until then, both old and new live alongside.

## Testing Decisions

- What makes a good test in this area (test through the deepened interface, not internals)
- Which modules get tests
- Prior art (similar tests in the codebase)
- Whether characterization tests are needed before refactor

## Out of Scope

Explicit. The most common refactor failure is scope creep; this section exists to push back.

## Risks & Mitigations

| Risk | Mitigation |
|---|---|
| <e.g., callers break> | <e.g., adapter shim during transition; deprecate after> |
```

---

## Claude-Code Refactoring Best Practices

When executing a refactor plan with Claude Code, follow these practices to keep changes safe and reversible:

1. **One behavior-preserving step per commit.** If a commit changes both behavior and structure, split it.
2. **Run tests between every commit.** A green checkpoint is what makes the next step safe.
3. **Use the Edit tool with exact strings** for precise changes — not regex search-and-replace, which can match unintended sites.
4. **Capture current behavior with a characterization test before refactoring** when the area is under-tested. Otherwise you have no oracle for "still works."
5. **Don't refactor and add features in the same commit.** They're separate intents. Refactor first, then add the feature on top of the cleaner ground.
6. **Verify scope didn't sprawl.** After each commit, run `git diff --stat HEAD~1` and confirm the file list matches what you planned. Surprise files = surprise scope.
7. **When in doubt, branch and squash-merge.** A messy branch with 30 tiny commits can squash-merge cleanly to `main` with one polished message.
8. **Stop and ask before destructive operations** — file deletions, function removals, large reorganizations. Show the planned change first.
9. **Don't bypass hooks** (`--no-verify`) to push refactors. If a hook fails, the refactor introduced a real problem.
10. **After the refactor lands, run `/check-production`** if the changed area is on the critical path. Refactors are where regressions sneak in.

---

## Rules

- **Use the glossary vocabulary** consistently — both project domain terms and the architecture vocabulary in [LANGUAGE.md](LANGUAGE.md).
- **Apply the deletion test** to anything that looks shallow.
- **Hammer scope explicitly** in the plan. Out-of-scope items are a feature, not a defect.
- **Tiny commits, working state.** Always.
- **Write the plan to `.claude/refactor-plan.md`** — recommend the user commit it.
- **Don't execute the refactor in this skill.** That's `/build-feature` (for feature refactors) or direct work after the plan is approved. This skill produces the plan.

## If Something Goes Wrong

- **No obvious refactor opportunities found** — report that the codebase looks reasonably clean and suggest a specific area for the user to point you to.
- **Refactor plan too large to execute safely** — break it into independent sub-plans with no shared state changes; tackle one at a time.
- **Working tree is dirty** — stop and ask the user to commit or stash before proceeding; do not plan a refactor on top of unstaged changes.