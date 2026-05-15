---
name: postmortem
description: MUST BE USED after a production outage, severe bug, data incident, or security exposure. Generates a structured blameless postmortem with an incident timeline, root cause analysis (5 Whys), contributing factors, and a concrete action item list. Trigger on `/postmortem`, "write postmortem", "incident review", "post-incident", "what happened", "root cause analysis", "RCA".
---

# /postmortem

You generate a structured blameless postmortem after a production incident. Blameless means the focus is on systems and processes — not individuals.

## Pre-flight

- Read `.claude/progress.md` (last 5 entries) and `.claude/context.md` if present
- Call `project-state-detector`; if mode is off-pattern for this skill, surface a one-line warning (do NOT block)

## Post-flight

- Append to `.claude/progress.md`: timestamp, skill name, output path, key decisions, suggested next step
- If `.claude/progress.md` is missing, create it with a header first

## Important

- Establish the incident timeline before analyzing root cause — jumping to "why" before "what happened" produces incorrect RCAs.
- Blameless means focusing on systems and processes, not individuals; do not name people in root cause or contributing factors.
- Action items must have owners and deadlines; postmortems without concrete next steps do not prevent recurrence.

## When to Use

- After a production outage (service unavailable > 5 minutes)
- After a data loss or data corruption incident
- After a security breach or credential exposure
- After a severe bug that affected multiple paying users or caused data inconsistency
- After any incident where the team asks "how do we prevent this next time?"

## Procedure

### Step 1: Gather the facts

Ask:
1. **What happened?** (symptoms visible to users)
2. **When did it start and end?** (timestamps, ideally UTC)
3. **Impact**: how many users affected? Any revenue impact? Data affected?
4. **Detection**: how was it found? (Sentry alert, user report, monitoring, someone noticed)
5. **Resolution**: what fixed it? (rollback, hotfix, env var change, manual DB fix)

### Step 2: Reconstruct the timeline

Work with the user to build a minute-by-minute timeline from first symptom to resolution. Pull timestamps from:
- Sentry event timestamps
- Vercel deployment logs (or Railway/Render if that's the platform)
- Git commit timestamps
- Slack/team communication timestamps

### Step 3: Root cause analysis (5 Whys)

Walk the 5 Whys starting from the immediate cause:

```
Immediate cause: [what directly caused the failure]
Why did that happen? → [reason 1]
Why did that happen? → [reason 2]
Why did that happen? → [reason 3]
Why did that happen? → [reason 4]
Why did that happen? → [root cause]
```

Distinguish:
- **Immediate cause** — what directly broke
- **Root cause** — why the immediate cause was possible in the first place
- **Contributing factors** — what made it worse or harder to detect

### Step 4: Generate the postmortem document

Produce the document using the template below. Save to `.claude/postmortem-<YYYY-MM-DD>-<short-slug>.md`.

### Step 5: Review action items

For each action item, confirm:
- It addresses a root cause or contributing factor (not just the symptom)
- It has a specific owner and due date
- It's in the team's backlog

Offer:
> Want me to run `/triage` on any of the bug-related action items to plan the fix?

---

## Postmortem Template

```markdown
# Postmortem — <incident title>
Date: <YYYY-MM-DD>
Severity: <SEV1 — full outage | SEV2 — degraded service | SEV3 — minor impact>
Status: <Draft | Under review | Final>

## Summary
<2–3 sentences a non-technical stakeholder can read. What happened, what broke, how it was resolved.>

## Impact
- **Duration**: <start time UTC> – <end time UTC> (<N minutes/hours>)
- **Users affected**: <count or estimate>
- **Revenue impact**: <$ or "none">
- **Data affected**: <what data, if any; "none" if clean>

---

## Timeline (UTC)

| Time | Event |
|---|---|
| HH:MM | First symptom observed |
| HH:MM | Detected by <alert / user report / monitoring> |
| HH:MM | <investigation step> |
| HH:MM | <hypothesis formed> |
| HH:MM | Root cause identified |
| HH:MM | Fix applied (<what the fix was>) |
| HH:MM | Service restored / regression confirmed resolved |
| HH:MM | Incident closed |

---

## Root cause analysis

### Immediate cause
<what directly caused it — one sentence>

### Root cause
<why the immediate cause was possible — the underlying system/process gap>

### Contributing factors
- <factor that made it worse or harder to detect>
- <factor>

### 5 Whys
1. Why did X fail? → Because Y
2. Why did Y happen? → Because Z
3. Why did Z happen? → Because ...
4. ...
5. Root cause: ...

---

## What went well
- <something that helped contain or speed resolution>
- <e.g., Sentry alert fired within 30 seconds; rollback took 2 minutes>

## What went poorly
- <something that made it worse or slower>
- <e.g., no staging environment to test the migration; alert threshold was too high>

---

## Action items

| # | Action | Addresses | Owner | Due | Status |
|---|---|---|---|---|---|
| 1 | <fix or process change> | <root cause / contributing factor> | @name | <date> | Open |
| 2 | | | | | Open |
| 3 | | | | | Open |
```

---

## Severity guide

| Severity | Meaning |
|---|---|
| SEV1 | Full outage — all users affected, core functionality unavailable |
| SEV2 | Degraded service — significant subset of users or key features affected |
| SEV3 | Minor impact — small subset of users, workaround available, no data loss |

---

## Rules

- **Blameless** — name systems, processes, and decisions, never individuals. "The deploy process lacked a staging gate" not "Alex deployed without testing."
- **Action items address root causes**, not symptoms. "Add a test for nil user in payment handler" is better than "be more careful."
- **Every action item needs an owner and a due date** — otherwise it won't happen.
- **Publish within 48 hours** — the longer you wait, the less useful the timeline details are.
- **Share with the team** — postmortems are most valuable when read by engineers who weren't involved in the incident.
- **File action items in the project backlog** — a postmortem with action items that stay in a doc and never ship is just documentation.

## If Something Goes Wrong

- **Incident timeline cannot be reconstructed** — pull logs from Sentry, PostHog, and the hosting platform; if logs were not retained, document the gap and add log retention as an action item.
- **Action items are disputed** — document dissenting views inline; do not remove action items to achieve consensus — unresolved disagreement is itself a risk worth documenting.
- **Postmortem is blocked on unavailable participants** — proceed with available information and mark sections as "Pending input from [role]"; do not delay indefinitely.