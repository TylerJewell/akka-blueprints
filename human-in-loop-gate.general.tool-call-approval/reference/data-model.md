# Data model

Every record, event, enum, and view row the generated system defines.

## `ToolRequest` (ToolRequestEntity state + ToolRequestsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Request id; equals the workflow id and entity id |
| `goal` | `Optional<String>` | yes | The submitted user goal |
| `status` | `ToolRequestStatus` | no | Lifecycle status |
| `plannedAt` | `Optional<Instant>` | yes | When the plan was recorded |
| `toolName` | `Optional<String>` | yes | Name of the tool selected by PlannerAgent |
| `parameters` | `Optional<String>` | yes | JSON object string of original call arguments |
| `rationale` | `Optional<String>` | yes | PlannerAgent's reasoning for the selection |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverNote` | `Optional<String>` | yes | Approver note |
| `editedParameters` | `Optional<String>` | yes | Operator-edited parameters JSON (overrides original when present) |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `executedAt` | `Optional<String>` | yes | ISO-8601 execution time |
| `executionOutput` | `Optional<String>` | yes | Result returned by ExecutorAgent |

Every nullable lifecycle field is `Optional<T>` because `ToolRequest` is the view row type (Lesson 6). `emptyState()` returns `ToolRequest.initial("")` with no `commandContext()` reference (Lesson 3).

## `ToolRequestStatus` enum

`PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `EXECUTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `ToolCallPlanned` | `recordPlan` after PlannerAgent returns | sets `plannedAt`, `toolName`, `parameters`, `rationale`, status → `PENDING_APPROVAL` |
| `ToolCallApproved` | `approve` command (operator via API) | sets `approvedAt`, `approvedBy`, `approverNote`, `editedParameters` (if present), status → `APPROVED` |
| `ToolCallRejected` | `reject` command (operator via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `ToolCallExecuted` | `recordExecution` after ExecutorAgent returns | sets `executedAt`, `executionOutput`, status → `EXECUTED` |

## Domain records

- `ToolCallPlan(String toolName, String parameters, String rationale)` — PlannerAgent result.
- `ApprovalDecision(String approvedBy, String approverNote, Optional<String> editedParameters)` — approve payload.
- `ToolCallResult(String output, String executedAt)` — ExecutorAgent result.

## View row type

`ToolRequestsView` rows are `ToolRequest`. One query: `getAllRequests` → `SELECT * AS requests FROM tool_requests_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ApprovalTasks.java)

- `PLAN` — `Task.name(...).resultConformsTo(ToolCallPlan.class)`.
- `EXECUTE` — `Task.name(...).resultConformsTo(ToolCallResult.class)`.

## Effective parameters resolution

`executeStep` reads `ToolRequestEntity` to obtain the effective parameters. Resolution order:
1. If `editedParameters` is present and non-empty on the entity → use `editedParameters`.
2. Otherwise → use `parameters`.

This logic lives in `ApprovalWorkflow.executeStep`, not in `ExecutorAgent`, so the agent always receives a resolved, single parameters string.
