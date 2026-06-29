# Data model

Every record, event, enum, and view row the generated system defines.

## `Case` (CaseEntity state + CasesView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Case id; equals the workflow id and entity id |
| `entityId` | `Optional<String>` | yes | The entity being screened |
| `status` | `CaseStatus` | no | Lifecycle status |
| `screenedAt` | `Optional<Instant>` | yes | When the screening result was recorded |
| `findings` | `Optional<String>` | yes | Screening findings from ScreenerAgent (with citations) |
| `recommendation` | `Optional<String>` | yes | `PASS`, `REFER`, or `BLOCK` from ScreenerAgent |
| `approvedAt` | `Optional<Instant>` | yes | When the compliance officer approved |
| `approvedBy` | `Optional<String>` | yes | Officer identity |
| `approverNote` | `Optional<String>` | yes | Officer's approval note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Rejection reason |
| `closedAt` | `Optional<String>` | yes | ISO-8601 close time |
| `disposition` | `Optional<String>` | yes | `APPROVED` when case is closed |

Every nullable lifecycle field is `Optional<T>` because `Case` is the view row type (Lesson 6). `emptyState()` returns `Case.initial("")` with no `commandContext()` reference (Lesson 3).

## `CaseStatus` enum

`SCREENED`, `OFFICER_APPROVED`, `REJECTED`, `CLOSED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `CaseScreened` | `recordScreening` after ScreenerAgent returns | sets `screenedAt`, `findings`, `recommendation`, status → `SCREENED` |
| `CaseOfficerApproved` | `approve` command (compliance officer via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `OFFICER_APPROVED` |
| `CaseRejected` | `reject` command (compliance officer via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `CaseClosed` | `recordClose` after VerdictAgent returns | sets `closedAt`, `disposition`, status → `CLOSED` |

## Domain records

- `ScreeningResult(String entityId, String findings, String recommendation)` — ScreenerAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `ClosedCase(String caseId, String disposition, String closedAt)` — VerdictAgent result.

## View row type

`CasesView` rows are `Case`. One query: `getAllCases` → `SELECT * AS cases FROM cases_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ScreeningTasks.java)

- `SCREEN` — `Task.name(...).resultConformsTo(ScreeningResult.class)`.
- `CLOSE` — `Task.name(...).resultConformsTo(ClosedCase.class)`.

## PII token map (CaseEntity auxiliary state)

CaseEntity also holds a `Map<String, String> piiTokenMap` that maps each generated token (e.g., `NAME_1`) back to the original value. This map is populated by the PII sanitizer before `ScreenerAgent` runs and is used only by the compliance officer UI after approval. It is not included in `CasesView` rows and is never passed to any LLM.
