---
name: prd
description: MUST BE USED when the user wants to create a PRD or product requirements doc. Synthesizes the current conversation and codebase understanding into a comprehensive PRD with functional and non-functional requirements, success metrics, risks, and rollout plan. Writes to `.claude/prd.md`. Does NOT interview the user — just synthesizes what's already known. Do NOT use when a PRD already exists — open `.claude/prd.md` directly and edit it, or run `/refactor` to restructure it.
---

# /prd

Take the current conversation context and codebase understanding and produce a comprehensive PRD. Do NOT interview the user — synthesize what you already know. If something is unknown, write `<TBD>` and call it out at the end.

## Pre-flight routing

Before running the PRD synthesis, evaluate whether the user has enough context to produce a useful PRD:

- Is a specific customer or persona named?
- Is a concrete problem articulated?
- Is the scope of "what to build" clear?

If two or more of these are missing or vague, suggest `/discover` instead of proceeding. Phrase it as a suggestion, not a block:

> "Your inputs look thin for a PRD synthesis. I can either proceed with what's here, or run `/discover` to surface the problem before structuring it. Which do you prefer?"

Proceed with the PRD if the user confirms, regardless of input quality. The routing is a suggestion, never a gate.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Before You Start

- Have a clear description of what you're building — even a rough paragraph is enough context to synthesize a meaningful PRD.
- If the codebase is brand new or empty, note that in context — the PRD will focus on design intent rather than current state.

## Process

### Step 1: Explore

Explore the repo to understand the current state of the codebase, if you haven't already. Don't be exhaustive — just enough to ground the PRD's Implementation Decisions section.

### Step 2: Sketch the modules

Sketch the major modules you would build or modify. Actively look for opportunities to extract deep modules — small interfaces hiding meaningful complexity.

Check with the user that these modules match their expectations. Confirm which modules they want tests written for.

### Step 3: Write the PRD to `.claude/prd.md`

Use the template below. Where information isn't available, write `<TBD>` and add it to the `## Open Questions` section at the end. Do NOT invent details.

### Step 4: Hand off

Tell the user the PRD is at `.claude/prd.md` and recommend `/plan` next to break it into vertical slices.

## PRD Template

```markdown
# PRD: <Feature or Product Name>

*Status: Draft · Author: <user> · Last updated: <YYYY-MM-DD>*

## 1. Problem Statement

The problem from the user's perspective. Avoid talking about implementation here — just the pain or unmet need.

## 2. Solution

The solution from the user's perspective. What changes for them. Still no implementation language.

## 3. Personas & Scenarios

### Primary persona
- **Who**: <role, context, goals>
- **Scenario**: <a concrete day-in-the-life scenario where this feature matters>

### Secondary personas (if any)
- **Who**: <role>
- **Scenario**: <when they'd encounter this>

## 4. User Stories

A long, numbered list. Each in the format: *As a `<actor>`, I want `<feature>`, so that `<benefit>`.*

1. As a <actor>, I want <feature>, so that <benefit>
2. ...

This list should be extensive and cover all aspects of the feature, including edge cases (empty states, errors, permissions).

## 5. Functional Requirements

Numbered, traceable requirements describing **what the system does**. Each gets an ID for cross-referencing in tests and tickets.

| ID | Requirement |
|---|---|
| FR-1 | <The system shall ...> |
| FR-2 | <The system shall ...> |
| FR-3 | <The system shall ...> |

## 6. Non-Functional Requirements

What the system must be like to operate it. Quantify where possible.

| Category | Requirement |
|---|---|
| **Performance** | <e.g., 95th percentile API response < 300ms> |
| **Security** | <e.g., all user data encrypted at rest; ownership enforced server-side> |
| **Privacy** | <e.g., PII excluded from logs and analytics events> |
| **Accessibility** | <e.g., WCAG 2.1 AA for all new UI> |
| **Scalability** | <e.g., handle 10x current traffic without architectural change> |
| **Reliability** | <e.g., 99.9% monthly uptime; no data loss on deploy> |
| **Observability** | <e.g., key actions emit PostHog events; errors hit Sentry> |
| **Compatibility** | <e.g., supports last 2 versions of Chrome/Safari/Firefox; no IE> |

## 7. Success Metrics

Quantifiable. Each metric has a target and a measurement source.

| Metric | Target | Source |
|---|---|---|
| <e.g., feature adoption> | <e.g., 20% of MAU within 30 days of launch> | <e.g., PostHog event `<event_name>`> |
| <e.g., conversion lift> | <e.g., +5% on paid signup> | <e.g., Stripe + PostHog funnel> |

## 8. Implementation Decisions

What we've decided to do, at a level of detail that survives refactors. Describe modules, contracts, and behaviors — not file paths or code snippets.

- **Modules to add or modify**: <module name + one-sentence purpose>
- **Interface changes**: <new public APIs, breaking changes>
- **Schema changes**: <new tables, columns, indexes — describe shape, not migration syntax>
- **API contracts**: <new endpoints, payload shape, auth requirements>
- **Architectural decisions**: <e.g., "use a webhook for X rather than polling because ...">
- **Specific interactions**: <subtle behaviors not captured in the user stories>

Do NOT include specific file paths or code snippets. They go stale fast.

## 9. Testing Decisions

How we'll verify this works.

- **What makes a good test here**: <e.g., test through the API, not internals; mock only at the third-party boundary>
- **Modules with tests**: <module 1, module 2, ...>
- **Prior art**: <similar tests in the codebase to mirror>
- **Manual test plan**: <bullet list a human can walk through end-to-end>

## 10. Dependencies

What this requires from other systems or people.

- **Third-party services**: <e.g., Stripe Customer Portal must be configured before launch>
- **Internal teams**: <e.g., needs design review by ...>
- **Data sources**: <e.g., relies on existing `users.tier` column>
- **Blocking work**: <other PRs/features that must land first>

## 11. Risks & Mitigations

| Risk | Impact | Likelihood | Mitigation |
|---|---|---|---|
| <e.g., webhook reliability> | <high — could miss subscription updates> | <medium> | <e.g., idempotency table + retry on 5xx> |
| <e.g., scope creep> | <medium — slips ship date> | <medium> | <e.g., explicit out-of-scope list below> |

## 12. Rollout Plan

How this ships.

- **Staged release**: <e.g., feature-flag behind `<flag-name>` for first 7 days>
- **Beta cohort**: <e.g., internal users only, then 10% of paid users>
- **Rollback plan**: <e.g., flag flip; no destructive migrations>
- **Comms**: <e.g., changelog entry, in-app announcement, email to affected users>
- **Verification post-launch**: <metrics to watch in the first 24/72 hours>

## 13. Out of Scope

Explicit list of things we are NOT doing in this PRD. Anything users might assume is included but isn't.

- <thing 1>
- <thing 2>

## 14. Open Questions

Things that need answers before `/plan` can produce a usable plan.

- <question 1>
- <question 2>

## 15. Further Notes

Anything else relevant — links to research, prior incidents, related ADRs.
```

## Rules

- **Synthesize, don't interview.** If you don't know something, write `<TBD>` and add it to Open Questions. Don't pad with imagined details.
- **Quantify NFRs and success metrics.** Vague targets like "fast" or "secure" are not requirements — they're hopes.
- **Implementation Decisions should survive refactors.** Use module names and behaviors, not file paths.
- **Write to `.claude/prd.md`**, not GitHub. Recommend the user commit it.
- **Recommend `/plan` next** in the handoff message.

## If Something Goes Wrong

- **Context too thin to synthesize requirements** — ask the user for a one-paragraph description of the problem and target user before proceeding; do not guess at requirements.
- **Conflicting signals between conversation and codebase** — surface the conflict explicitly in the PRD under a "Assumptions & Open Questions" section; do not silently pick one.
- **`.claude/prd.md` write fails** — confirm write permission to `.claude/`; create the directory if needed.