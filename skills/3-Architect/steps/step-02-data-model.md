# Step 02 — Data Model

**Goal:** Decide what data entities exist, how they relate, and what the schema looks like at a high level.

## Instructions

Read `.claude/prd.md` if it exists. Identify the domain entities implied by the requirements.

For each entity, determine:
- Name and purpose
- Key fields (not exhaustive — high-level)
- Relationships to other entities (1:1, 1:N, N:M)
- Any indexing or query pattern constraints

Ask the PM to review and correct the proposed model before continuing.

Specific questions:

1. Are there any existing tables or models I should build on rather than create new ones?
2. Are there any fields that must be encrypted at rest?
3. Are there any multi-tenancy constraints? (e.g., all records must be scoped to an `org_id`)
4. What are the most frequent read queries? (to decide indexes early)

## Output

Append to `.claude/architecture.md`:

```markdown
## Data Model (Step 02)

### Entities

**<EntityName>**
- Purpose: <one-line description>
- Key fields: <field: type, ...>
- Relationships: <related entity: cardinality>
- Notes: <constraints, encryption, tenancy>

[repeat for each entity]

### Query patterns
<High-frequency queries that drive index decisions>
```

## Checkpoint

Present the data model and ask: "Does this data model match your intent? Ready to proceed to integrations?"
