# Data model

Every Java record the generated system defines. Nullable lifecycle fields are `Optional<T>` (Lesson 6) so the `ActionsView` materializer accepts the row.

## `Action` (event-sourced state and View row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Action / workflow id (UUID) |
| `request` | `Optional<String>` | yes | Original user request |
| `status` | `ActionStatus` | no | Lifecycle status |
| `plannedAt` | `Optional<Instant>` | yes | When the plan was recorded |
| `toolName` | `Optional<String>` | yes | Proposed tool |
| `toolArguments` | `Optional<String>` | yes | Proposed arguments (JSON-style string) |
| `rationale` | `Optional<String>` | yes | Why this tool/arguments |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver id |
| `approverComment` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter id |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `executedAt` | `Optional<Instant>` | yes | When the tool ran |
| `toolOutput` | `Optional<String>` | yes | Tool result string |
| `escalatedAt` | `Optional<Instant>` | yes | When escalated |

`Action.initial(String id, String request)` returns a row with `status = CREATED` and every `Optional` empty. No `commandContext()` reference in `emptyState()` (Lesson 3).

## `ActionStatus` (enum)

`CREATED · PLANNED · APPROVED · REJECTED · EXECUTED · ESCALATED`

## `ActionEvent` (sealed interface, 5 variants)

| Event | Trigger |
|---|---|
| `ActionPlanned(actionId, toolName, toolArguments, rationale, timestamp)` | `ApprovalWorkflow.planStep` records the agent's plan |
| `ActionApproved(actionId, approvedBy, comment, timestamp)` | `POST /api/actions/{id}/approve` |
| `ActionRejected(actionId, rejectedBy, reason, timestamp)` | `POST /api/actions/{id}/reject` |
| `ActionExecuted(actionId, toolOutput, timestamp)` | `ApprovalWorkflow.executeStep` records the tool result |
| `ActionEscalated(actionId, timestamp)` | `StuckActionMonitor` marks a stale `PLANNED` action |

## Auxiliary records (in `application/`)

```java
record ActionPlan(String toolName, String toolArguments, String rationale) {}
record ToolResult(String toolOutput, String executedAt) {}
record ApprovalDecision(String approvedBy, String comment) {}
```

## InboundRequestQueue

Single-instance entity. One command `enqueueRequest(String request)` emitting `InboundRequestQueued(request, timestamp)`. `RequestConsumer` subscribes and starts an `ApprovalWorkflow`.

## View row type

`ActionsView` row type is the `Action` record above. One query: `getAllActions` → `SELECT * AS actions FROM actions_view`. No status filter in the query — callers filter client-side.
