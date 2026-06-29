# Data model

Every record, event, enum, and view row the generated system defines.

## `Action` (ActionEntity state + ActionsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Action id; equals the workflow id and entity id |
| `operationRequest` | `Optional<String>` | yes | The submitted operation request text |
| `status` | `ActionStatus` | no | Lifecycle status |
| `proposedAt` | `Optional<Instant>` | yes | When the proposal was recorded |
| `actionType` | `Optional<String>` | yes | Category of operation (e.g. SCALE, RESTART, ROTATE_KEY) |
| `target` | `Optional<String>` | yes | Specific resource or service the action targets |
| `rationale` | `Optional<String>` | yes | Agent's analysis of why the action is appropriate |
| `estimatedImpact` | `Optional<String>` | yes | Expected effect and risk assessment |
| `approvedAt` | `Optional<Instant>` | yes | When the operator approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverNote` | `Optional<String>` | yes | Optional approver note |
| `deniedAt` | `Optional<Instant>` | yes | When the operator denied |
| `deniedBy` | `Optional<String>` | yes | Denier identity |
| `denyReason` | `Optional<String>` | yes | Denial reason |
| `executedAt` | `Optional<String>` | yes | ISO-8601 execution completion time |
| `outcome` | `Optional<String>` | yes | Execution outcome (SUCCESS / PARTIAL / FAILED with description) |
| `executionDetails` | `Optional<String>` | yes | Narrative of what the execution step did |

Every nullable lifecycle field is `Optional<T>` because `Action` is the view row type (Lesson 6). `emptyState()` returns `Action.initial("")` with no `commandContext()` reference (Lesson 3).

## `ActionStatus` enum

`PROPOSED`, `APPROVED`, `DENIED`, `EXECUTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ActionProposed` | `recordProposal` after ProposalAgent returns | sets `proposedAt`, `actionType`, `target`, `rationale`, `estimatedImpact`; status → `PROPOSED` |
| `ActionApproved` | `approve` command (operator via API) | sets `approvedAt`, `approvedBy`, `approverNote`; status → `APPROVED` |
| `ActionDenied` | `deny` command (operator via API) | sets `deniedAt`, `deniedBy`, `denyReason`; status → `DENIED` |
| `ActionExecuted` | `recordExecution` after ExecutionAgent returns | sets `executedAt`, `outcome`, `executionDetails`; status → `EXECUTED` |

## Domain records

- `ActionProposal(String actionType, String target, String rationale, String estimatedImpact)` — ProposalAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `ActionResult(String outcome, String completedAt, String details)` — ExecutionAgent result.

## View row type

`ActionsView` rows are `Action`. One query: `getAllActions` → `SELECT * AS actions FROM actions_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ApprovalTasks.java)

- `PROPOSE` — `Task.name(...).resultConformsTo(ActionProposal.class)`.
- `EXECUTE` — `Task.name(...).resultConformsTo(ActionResult.class)`.
