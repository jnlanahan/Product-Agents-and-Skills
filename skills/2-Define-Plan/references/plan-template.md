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

> **Phase key** — slices are organized in build order:
> - **Phase A — Working Prototype:** Core thing working end-to-end. Mocked data is fine. Something demoable. Build this first.
> - **Phase B — Minimum Functionality:** Real persistence, basic validation, minimum happy path. Something a user can actually test.
> - **Phase C — Production Hardening:** Auth, security, payments, monitoring. Build this last. Omit if `WORKFLOW_SCOPE` is `PROTOTYPE`, `PERSONAL`, or `ADD_FEATURE` (unless capability already exists).

---

### Phase A — Working Prototype

*Goal: get the core behavior running end-to-end with mocked or minimal data. Something demoable at the end of this phase.*

#### 4.A.1  <Slice A1 Name — Tracer Bullet>

**User-facing behavior:** <one sentence — what can the user do?>
**Demoable outcome:** <what they see when this slice is complete; mocked data acceptable>
**Depends on:** None

##### 4.A.1.1  Schema
<Tables / columns / indexes needed for the tracer bullet, or "no change.">

##### 4.A.1.2  Storage
<Minimal data-access functions — stub or real, whichever is faster to demo.>

##### 4.A.1.3  Routes / Actions
<Endpoint(s) or server action(s) that make the core behavior work.>

##### 4.A.1.4  Client
<Hook(s) + component(s) that connect UI to the route.>

##### 4.A.1.5  Tests
- RED → GREEN: <core behavior 1>
- RED → GREEN: <core behavior 2>

*One test → one impl per cycle. Mock at system boundaries only.*

##### 4.A.1.6  Migration
<None / or sketch>

##### 4.A.1.7  Verification
- [ ] All Phase A.1 tests green
- [ ] `npm run build` clean
- [ ] Manual: <demo step — show the thing working end-to-end>
- [ ] Deferred to Phase B: <persistence / validation items>

---

#### 4.A.2  <Slice A2 Name>

**User-facing behavior:** <one sentence>
**Demoable outcome:** <what they see>
**Depends on:** 4.A.1

##### 4.A.2.1–4.A.2.7  *(same subsections as 4.A.1)*

*(Repeat 4.A.x for each Phase A slice)*

---

### Phase B — Minimum Functionality

*Goal: replace mocks with real data, add basic validation, complete the minimum happy path. Something a tester can use.*

#### 4.B.1  <Slice B1 Name — Real Persistence>

**User-facing behavior:** <one sentence>
**Demoable outcome:** <data survives a page reload; error shows on bad input>
**Depends on:** 4.A.1 (or last Phase A slice)

##### 4.B.1.1  Schema
<Finalize schema — add columns deferred from Phase A.>

##### 4.B.1.2  Storage
<Replace stub storage with real DB calls. Add error handling.>

##### 4.B.1.3  Routes / Actions
<Add validation at API boundary (e.g. Zod). Return proper error shapes.>

##### 4.B.1.4  Client
<Wire up error states. Handle loading / empty states.>

##### 4.B.1.5  Tests
- RED → GREEN: <persistence — data survives round-trip>
- RED → GREEN: <validation — bad input rejected with correct error>
- RED → GREEN: <error handling — server error shows graceful UI>

##### 4.B.1.6  Migration
<Production-safe migrations deferred from Phase A.>

##### 4.B.1.7  Verification
- [ ] All Phase B.1 tests green
- [ ] `npm run build` clean
- [ ] Manual: <happy path from blank state through success state>
- [ ] Deferred to Phase C: <auth / security / payments>

---

#### 4.B.2  <Slice B2 Name>

**User-facing behavior:** <one sentence>
**Depends on:** 4.B.1

##### 4.B.2.1–4.B.2.7  *(same subsections)*

*(Repeat 4.B.x for each Phase B slice)*

---

### Phase C — Production Hardening

*Goal: auth, security boundaries, payments, monitoring. Build after Phase A + B are fully working. Remove this entire phase block if `WORKFLOW_SCOPE` excludes it.*

#### 4.C.1  <Slice C1 Name — Auth / Security>

**User-facing behavior:** <e.g. "Users log in and see only their own data">
**Demoable outcome:** <two users cannot see each other's data; session persists>
**Depends on:** last Phase B slice

##### 4.C.1.1  Schema
<User / session tables; ownership columns on feature tables.>

##### 4.C.1.2  Storage
<All queries scoped to `userId`. Row-level filtering.>

##### 4.C.1.3  Routes / Actions
<All protected routes check session. Auth middleware.>

##### 4.C.1.4  Client
<Auth-gated pages. Redirect unauthenticated users.>

##### 4.C.1.5  Tests
- RED → GREEN: <unauthenticated request → 401/redirect>
- RED → GREEN: <User B cannot read User A's resource → 403/404>
- RED → GREEN: <session persists across reload>

##### 4.C.1.6  Migration
<Add `userId` foreign keys; backfill or seed if needed.>

##### 4.C.1.7  Verification
- [ ] All Phase C.1 tests green
- [ ] `npm run build` clean
- [ ] Manual: two-browser test — User A's data not visible to User B

---

#### 4.C.2  <Slice C2 Name — Payments / Billing>  *(omit if not applicable)*

**Depends on:** 4.C.1

##### 4.C.2.1–4.C.2.7  *(same subsections)*

---

#### 4.C.3  <Slice C3 Name — Monitoring / Observability>  *(omit if not applicable)*

**User-facing behavior:** Error tracking and usage analytics active in production.
**Depends on:** 4.C.1

##### 4.C.3.1–4.C.3.7  *(same subsections)*

---

*(Repeat 4.C.x for each Phase C slice. Remove the entire Phase C block if `WORKFLOW_SCOPE` excludes it.)*

---

## 5.0  Delivery Schedule

| #   | Milestone | Phase / Slice(s) | Status |
|-----|-----------|------------------|--------|
| 5.1 | Core behavior demoable | Phase A (4.A.1–4.A.n) | Not started |
| 5.2 | Minimum functionality testable | Phase B (4.B.1–4.B.n) | Not started |
| 5.3 | Production-hardened | Phase C (4.C.1–4.C.n) | Not started |

*Remove rows for phases excluded by `WORKFLOW_SCOPE`.*

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
