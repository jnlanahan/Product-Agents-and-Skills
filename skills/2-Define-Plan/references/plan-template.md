# Plan: <Feature or Product Name>

**Status:** Draft
**Source:** `.claude/prd.md` *(or "current conversation")*
**Created:** <YYYY-MM-DD>
**Last Updated:** <YYYY-MM-DD>

---

## 1.0  Overview

### 1.1  Purpose
<One sentence: what this plan delivers and why.>

### 1.2  Background
<What triggered this work — user problem, PRD summary, or prior context.>

### 1.3  Objectives
- <Objective 1>
- <Objective 2>

### 1.4  Success Criteria

| #     | Criterion | How to verify | Threshold |
|-------|-----------|---------------|-----------|
| 1.4.1 | <observable outcome, e.g. "User can create a note and see it in the list"> | Manual: create → reload → confirm | Must work in Chrome + Safari |
| 1.4.2 | <permission boundary, e.g. "User B cannot read User A's data"> | API test: GET with wrong token → 403/404 | Zero leakage |
| 1.4.3 | <data integrity, e.g. "Deleting an item removes it from the DB"> | DB query after delete → row gone | Row must not exist |
| 1.4.4 | <error handling, e.g. "Empty required field shows inline error"> | UI: submit blank form → error shown before server call | Error visible, no server call |

*These assertions are written before implementation and verified by a fresh reviewer who has not seen the code.*

---

## 2.0  Scope

### 2.1  In Scope
- <item>

### 2.2  Out of Scope
- <item explicitly deferred>

### 2.3  Assumptions
- <assumption that, if wrong, would change the plan>

### 2.4  Constraints
- <technical or resource constraint>

---

## 3.0  Architecture & Decisions

### 3.1  Stack & Layering
**Framework:** <detected>
**DB / ORM:** <detected>
**Test framework:** <detected>
**Layer order for this project:** schema → storage → routes → hooks → components *(adjust to match actual project)*

### 3.2  Cross-Cutting Rules
- **Validation:** <e.g., Zod at every API boundary, drizzle-zod for DB shape>
- **Auth / Ownership:** <e.g., every protected route checks `userId === resource.userId` server-side>
- **Error handling:** <match existing pattern from pattern-finder, e.g. `throw new AppError(...)`>
- **Naming:** <follow existing convention, e.g. `useNotes`, `noteRoutes.ts`>

### 3.3  Key Decisions

| #     | Decision | Rationale | Alternative considered |
|-------|----------|-----------|------------------------|
| 3.3.1 | | | |

---

## 4.0  Work Breakdown

### 4.1  <Slice 1 Name — Tracer Bullet>

**User-facing behavior:** <one sentence>
**Demoable outcome:** <what the user can see/do when this slice is complete>
**Depends on:** None

#### 4.1.1  Schema
<New tables / columns / indexes, or "no change.">

#### 4.1.2  Storage
<New data-access functions, or "no change.">

#### 4.1.3  Routes / Actions
<New endpoints or server actions, or "no change.">

#### 4.1.4  Client
<New hooks + components, or "no change.">

#### 4.1.5  Tests
- RED → GREEN: <observable behavior 1 — should fail because X>
- RED → GREEN: <observable behavior 2>
- RED → GREEN: <permission / ownership boundary>

*One test → one impl per cycle. Mock only at system boundaries (DB, third-party APIs, time, randomness).*

#### 4.1.6  Migration
<None / or sketch: add column X to table Y, nullable first then backfill>

#### 4.1.7  Verification
- [ ] All tests green
- [ ] `npm run build` clean
- [ ] Manual: <demo step>
- [ ] Out of scope for this slice: <items deferred to later slices>

---

### 4.2  <Slice 2 Name>

**User-facing behavior:** <one sentence>
**Demoable outcome:** <what the user can see/do>
**Depends on:** 4.1

#### 4.2.1  Schema
#### 4.2.2  Storage
#### 4.2.3  Routes / Actions
#### 4.2.4  Client
#### 4.2.5  Tests
#### 4.2.6  Migration
#### 4.2.7  Verification

*(Repeat 4.x.1 – 4.x.7 for each additional slice)*

---

## 5.0  Delivery Schedule

| #   | Milestone | Slice(s) | Status |
|-----|-----------|----------|--------|
| 5.1 | Tracer bullet live | 4.1 | Not started |
| 5.2 | Write path complete | 4.2 | Not started |
| 5.3 | Full happy path | 4.3 | Not started |
| 5.4 | Edge cases & permissions | 4.4 | Not started |

---

## 6.0  Open Questions

| #   | Question | Owner | Due |
|-----|----------|-------|-----|
| 6.1 | <question blocking a slice> | <person> | <date> |

---

## 7.0  Risks

| #   | Risk | Likelihood | Impact | Mitigation |
|-----|------|------------|--------|------------|
| 7.1 | | Low / Med / High | Low / Med / High | |
