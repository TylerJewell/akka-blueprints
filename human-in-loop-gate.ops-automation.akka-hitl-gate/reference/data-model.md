# Data model

Every record, event, enum, and view row the generated system defines.

## `Action` (ActionEntity state + ActionsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Action id; equals the workflow id and entity id |
| `taskDescription` | `Optional<String>` | yes | The submitted ops task description |
| `status` | `ActionStatus` | no | Lifecycle status |
| `proposedAt` | `Optional<Instant>` | yes | When the proposal was recorded |
| `proposalSummary` | `Optional<String>` | yes | Concise description of the proposed action |
| `estimatedImpact` | `Optional<String>` | yes | Systems affected and scope |
| `actionPayload` | `Optional<String>` | yes | Machine-readable action directive |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverNote` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `executedAt` | `Optional<String>` | yes | ISO-8601 execution time |
| `outcome` | `Optional<String>` | yes | Execution result description |

Every nullable lifecycle field is `Optional<T>` because `Action` is the view row type (Lesson 6). `emptyState()` returns `Action.initial("")` with no `commandContext()` reference (Lesson 3).

## `ActionStatus` enum

`PROPOSED`, `APPROVED`, `REJECTED`, `EXECUTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ActionProposed` | `recordProposal` after ProposalAgent returns | sets `proposedAt`, `proposalSummary`, `estimatedImpact`, `actionPayload`, status → `PROPOSED` |
| `ActionApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `ActionRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `ActionExecuted` | `recordExecution` after ExecutionAgent returns | sets `executedAt`, `outcome`, status → `EXECUTED` |

## Domain records

- `ActionProposal(String summary, String estimatedImpact, String actionPayload)` — ProposalAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `ActionResult(String outcome, String executedAt)` — ExecutionAgent result.

## View row type

`ActionsView` rows are `Action`. One query: `getAllActions` → `SELECT * AS actions FROM actions_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (GateTasks.java)

- `PROPOSE` — `Task.name(...).resultConformsTo(ActionProposal.class)`.
- `EXECUTE` — `Task.name(...).resultConformsTo(ActionResult.class)`.
