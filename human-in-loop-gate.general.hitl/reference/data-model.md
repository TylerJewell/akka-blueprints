# Data model

Every record, event, enum, and view row the generated system defines.

## `Action` (ActionEntity state + ActionsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Action id; equals the workflow id and entity id |
| `requestText` | `Optional<String>` | yes | The submitted request text |
| `status` | `ActionStatus` | no | Lifecycle status |
| `proposedAt` | `Optional<Instant>` | yes | When the proposal was recorded |
| `proposalSummary` | `Optional<String>` | yes | One-to-two sentence description of the proposed action |
| `proposalRationale` | `Optional<String>` | yes | Reasoning behind the proposal |
| `proposalRiskLevel` | `Optional<String>` | yes | `LOW`, `MEDIUM`, or `HIGH` |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverComment` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `executedAt` | `Optional<String>` | yes | ISO-8601 execution time |
| `executionOutcome` | `Optional<String>` | yes | Description of what was done |

Every nullable lifecycle field is `Optional<T>` because `Action` is the view row type (Lesson 6). `emptyState()` returns `Action.initial("")` with no `commandContext()` reference (Lesson 3).

## `ActionStatus` enum

`PROPOSED`, `APPROVED`, `REJECTED`, `EXECUTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ActionProposed` | `recordProposal` after ProposalAgent returns | sets `proposedAt`, `proposalSummary`, `proposalRationale`, `proposalRiskLevel`, status → `PROPOSED` |
| `ActionApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverComment`, status → `APPROVED` |
| `ActionRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `ActionExecuted` | `recordExecution` after ExecutionAgent returns | sets `executedAt`, `executionOutcome`, status → `EXECUTED` |

## Domain records

- `ActionProposal(String summary, String rationale, String riskLevel)` — ProposalAgent result.
- `ApprovalDecision(String approvedBy, String comment)` — approve payload.
- `ActionResult(String outcome, String completedAt)` — ExecutionAgent result.

## View row type

`ActionsView` rows are `Action`. One query: `getAllActions` → `SELECT * AS actions FROM actions_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ActionTasks.java)

- `PROPOSE` — `Task.name(...).resultConformsTo(ActionProposal.class)`.
- `EXECUTE` — `Task.name(...).resultConformsTo(ActionResult.class)`.
