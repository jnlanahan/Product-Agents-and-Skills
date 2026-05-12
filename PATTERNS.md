# PATTERNS.md — Common Workflow Sequences

Reference patterns for `project-state-detector` and PMs navigating the PDLC. These are documented sequences, not enforced workflows. No conformance required.

---

## Pattern 1: Greenfield MVP

**When:** Brand new product, starting from scratch.

**Sequence:**
```
/start → /discover → /prd → /architect → /plan → /build-feature → /check-production → /deploy
```

**PDLC phases touched:** All 7.

**Detection signals:** No `.claude/prd.md`, no code, `.claude/context.md` just created.

---

## Pattern 2: New SaaS Feature

**When:** Adding a scoped feature to a live production codebase.

**Sequence:**
```
/prd → /measure → /plan → /build-feature → /feature-flag → /check-production → /deploy → /post-launch-review
```

**PDLC phases touched:** Define, Build, Validate, Deploy, Learn.

**Detection signals:** `.claude/prd.md` exists, active git history, no `.claude/plan.md` for new feature.

---

## Pattern 3: Brownfield Refactor

**When:** Paying down technical debt without changing behavior.

**Sequence:**
```
/architect → /refactor → /plan → /build-feature (with pattern-finder heavy) → /check-production
```

**PDLC phases touched:** Design, Define, Build, Validate.

**Detection signals:** `.claude/prd.md` may not exist, active git history, user mentions "refactor", "cleanup", "modernize".

---

## Pattern 4: Bug Sprint

**When:** Addressing a set of production bugs or incidents.

**Sequence:**
```
/triage → /build-feature (per bug) → /check-production → /postmortem (if severe)
```

**PDLC phases touched:** Validate, Build, Learn.

**Detection signals:** `.claude/bugs/` directory exists, recent `/triage` entries in `progress.md`.

---

## Pattern 5: Hardening Pass

**When:** Pre-launch or periodic security/readiness audit.

**Sequence:**
```
/check-production → /build-feature (fixes) → /check-production (re-run) → /deploy
```

**PDLC phases touched:** Validate, Build, Deploy.

**Detection signals:** No deploy artifact yet, `/check-production` run recently, open Critical/High findings.

---

## Using these patterns

`project-state-detector` uses these patterns as signal, not prescription. When `.claude/` state matches a pattern with high confidence, it surfaces "you look like Pattern N" in its SIGNALS output. The PM is never forced to follow a pattern.

To add a new pattern: document it here in the same format, then note it in `AGENTS.md`.
