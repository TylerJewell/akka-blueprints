# Data model

Every record, event, enum, and view row the generated system defines.

## `Case` (CaseEntity state + CasesView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Case id; equals the workflow id and entity id |
| `customerRequest` | `Optional<String>` | yes | The submitted service request text |
| `status` | `CaseStatus` | no | Lifecycle status |
| `triagedAt` | `Optional<Instant>` | yes | When the triage result was recorded |
| `triageSummary` | `Optional<String>` | yes | One-to-two sentence issue description from TriageAgent |
| `proposedResolution` | `Optional<String>` | yes | Concrete resolution proposed by TriageAgent |
| `urgency` | `Optional<String>` | yes | `LOW`, `MEDIUM`, or `HIGH` assigned during triage |
| `approvedAt` | `Optional<Instant>` | yes | When the specialist approved |
| `approvedBy` | `Optional<String>` | yes | Specialist identity on approval |
| `specialistComment` | `Optional<String>` | yes | Specialist note on approval |
| `declinedAt` | `Optional<Instant>` | yes | When the specialist declined |
| `declinedBy` | `Optional<String>` | yes | Specialist identity on decline |
| `declineReason` | `Optional<String>` | yes | Reason given when declining |
| `fulfilledAt` | `Optional<String>` | yes | ISO-8601 fulfilment time |
| `confirmationId` | `Optional<String>` | yes | Unique confirmation reference from FulfillmentAgent |

Every nullable lifecycle field is `Optional<T>` because `Case` is the view row type (Lesson 6). `emptyState()` returns `Case.initial("")` with no `commandContext()` reference (Lesson 3).

## `CaseStatus` enum

`TRIAGED`, `APPROVED`, `DECLINED`, `FULFILLED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `CaseTriaged` | `recordTriage` after TriageAgent returns | sets `triagedAt`, `triageSummary`, `proposedResolution`, `urgency`, status → `TRIAGED` |
| `CaseApproved` | `approve` command (specialist via API) | sets `approvedAt`, `approvedBy`, `specialistComment`, status → `APPROVED` |
| `CaseDeclined` | `decline` command (specialist via API) | sets `declinedAt`, `declinedBy`, `declineReason`, status → `DECLINED` |
| `CaseFulfilled` | `recordFulfilment` after FulfillmentAgent returns | sets `fulfilledAt`, `confirmationId`, status → `FULFILLED` |

## Domain records

- `TriageResult(String summary, String proposedResolution, String urgency)` — TriageAgent result.
- `ApprovalDecision(String approvedBy, String comment)` — approve payload.
- `FulfilledCase(String confirmationId, String fulfilledAt)` — FulfillmentAgent result.

## View row type

`CasesView` rows are `Case`. One query: `getAllCases` → `SELECT * AS cases FROM cases_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ConciergeTasks.java)

- `TRIAGE` — `Task.name(...).resultConformsTo(TriageResult.class)`.
- `FULFIL` — `Task.name(...).resultConformsTo(FulfilledCase.class)`.
