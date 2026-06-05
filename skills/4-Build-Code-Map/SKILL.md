---
name: 4-Build-Code-Map
description: Use when the user is unfamiliar with a section of code, asks the agent to "zoom out", or wants a higher-level map of how an area fits into the bigger picture.
when_to_use: "User says 'what's going on here', 'explain this area', 'give me a code map', 'zoom out', 'how does this fit into the bigger picture'."
---

# /4-Build-Code-Map

I don't know this area of code well. Go up a layer of abstraction. Give me a map of all the relevant modules and callers, what they do at a glance, and how data flows between them. Surface the public interfaces; hide the internals.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Before You Start

- Specify the area of code to map — running this against the whole project produces an overwhelming result.
- This is a read-only orientation tool; no code changes will be made.

## If Something Goes Wrong

- **Area too large to map usefully** — ask the user to narrow the scope to one module, route, or component; map one layer at a time.
- **No clear entry points found** — start from the router/controller layer and trace inward rather than trying to find a top-level coordinator.
- **Circular dependencies confuse the map** — surface the cycles explicitly and flag them as a likely source of complexity before continuing the map.