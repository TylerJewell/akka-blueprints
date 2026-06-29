# Data model

Every record, event, enum, and view row the generated system defines.

## `Task` (TaskEntity state + TasksView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Task id; equals the workflow id and entity id |
| `request` | `Optional<String>` | yes | The submitted task request text |
| `status` | `TaskStatus` | no | Lifecycle status |
| `processedAt` | `Optional<Instant>` | yes | When the processed result was recorded |
| `taskSummary` | `Optional<String>` | yes | Short summary of what was analyzed |
| `taskRecommendation` | `Optional<String>` | yes | Actionable recommendation text |
| `approvedAt` | `Optional<Instant>` | yes | When approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverComment` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `finalizedAt` | `Optional<String>` | yes | ISO-8601 finalization time |
| `finalizedOutcome` | `Optional<String>` | yes | Outcome confirmation text |

Every nullable lifecycle field is `Optional<T>` because `Task` is the view row type (Lesson 6). `emptyState()` returns `Task.initial("")` with no `commandContext()` reference (Lesson 3).

## `TaskStatus` enum

`PROCESSED`, `APPROVED`, `REJECTED`, `FINALIZED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `TaskProcessed` | `recordProcessed` after TaskProcessorAgent returns | sets `processedAt`, `taskSummary`, `taskRecommendation`, status → `PROCESSED` |
| `TaskApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverComment`, status → `APPROVED` |
| `TaskRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `TaskFinalized` | `recordFinalization` after TaskFinalizerAgent returns | sets `finalizedAt`, `finalizedOutcome`, status → `FINALIZED` |

## Domain records

- `ProcessedTask(String summary, String recommendation)` — TaskProcessorAgent result.
- `ApprovalDecision(String approvedBy, String comment)` — approve payload.
- `FinalizedTask(String outcome, String finalizedAt)` — TaskFinalizerAgent result.

## View row type

`TasksView` rows are `Task`. One query: `getAllTasks` → `SELECT * AS tasks FROM tasks_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ApprovalTasks.java)

- `PROCESS` — `Task.name(...).resultConformsTo(ProcessedTask.class)`.
- `FINALIZE` — `Task.name(...).resultConformsTo(FinalizedTask.class)`.
