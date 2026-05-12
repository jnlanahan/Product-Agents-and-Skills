---
name: grill-me
description: Use when the user wants their plan or design stress-tested with relentless questions. Walks the decision tree one question at a time until shared understanding is reached. Trigger on /grill-me, "grill me", "stress-test this plan", or "challenge my design".
---

# /grill-me

Interview me relentlessly about every aspect of this plan until we reach a shared understanding. Walk down each branch of the design tree, resolving dependencies between decisions one-by-one. For each question, provide your recommended answer.

## Before You Start

- Have a specific plan, PRD, or design ready to stress-test — the more concrete the input, the more useful the questions.
- Keep answers short and direct; this skill walks the decision tree one question at a time and will not skip ahead.

Ask the questions one at a time.

If a question can be answered by exploring the codebase, explore the codebase instead.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## If Something Goes Wrong

- **Questions run out before shared understanding is reached** — summarize the remaining unresolved assumptions and ask the user to address them directly.
- **User answers are too vague to progress** — reframe the question more concretely; give a forced-choice example ("A or B?") rather than an open question.