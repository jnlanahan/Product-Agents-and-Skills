---
title: "Audit Output Format"
skill: "5-Validate-Production-Readiness"
---

# Audit Output Format

## Standard Structure

Wrap the `prod-readiness-auditor` output in this template when presenting results:

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

---

## Calibration by Classification

### Greenfield project

Audit will mostly find Mediums (no tests, no CI, missing nice-to-haves). That's expected. Don't overstate severity. Recommend running `/setup-project` if not done.

### Wired project (this template)

Audit should find few Criticals. Most findings will be specific things to harden (e.g., "rate limit endpoint X is unprotected"). Treat as a polish exercise.

### Vibe-coded project

Audit will surface real risks — committed secrets, missing webhook verification, no CI, no error tracking. Lead with rotation if secrets are exposed. The recommended fix order matters most here; sequencing prevents the user from doing things in dangerous orders (e.g., enabling Sentry source maps before rotating Sentry auth token in CI).
