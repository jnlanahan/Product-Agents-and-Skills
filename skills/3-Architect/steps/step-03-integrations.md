# Step 03 — Integrations

**Goal:** Surface every external system, third-party API, or internal service that this feature touches — and identify the constraints each introduces.

## Instructions

From the PRD and conversation context, enumerate:
- External APIs called (auth providers, payment processors, email, storage, AI providers)
- Internal services or microservices consumed
- Webhooks received or sent
- File systems, queues, caches touched

For each integration, assess:
- **Auth model**: API key, OAuth, service account, signed JWT?
- **Rate limits**: known limits that affect design?
- **Failure mode**: what happens if this integration goes down?
- **Data ownership**: who owns the data — us, the third party, both?

Ask the PM:
1. Are there any integrations not captured above?
2. Are there any integrations we must avoid for compliance or contractual reasons?
3. Which integrations are new (to be wired) vs. existing (to be extended)?

## Output

Append to `.claude/architecture.md`:

```markdown
## Integrations (Step 03)

| Integration | Type | Auth | Rate limits | Failure mode | New/Existing |
|---|---|---|---|---|---|
| <name> | API / webhook / storage | <type> | <limits> | <behavior> | new / existing |

**Constraints:**
<Any compliance, contractual, or architectural constraints from integrations>
```

## Checkpoint

Present the integrations table and ask: "Any integrations missing or wrong? Ready to proceed to tradeoffs?"
