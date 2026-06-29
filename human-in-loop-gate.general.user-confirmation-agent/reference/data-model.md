# Data model

Every record, event, enum, and view row the generated system defines.

## `Request` (RequestEntity state + RequestsView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Request id; equals the workflow id and entity id |
| `description` | `Optional<String>` | yes | The submitted request description |
| `status` | `RequestStatus` | no | Lifecycle status |
| `plannedAt` | `Optional<Instant>` | yes | When the action plan was recorded |
| `proposedActions` | `Optional<List<String>>` | yes | Ordered list of proposed action strings |
| `confirmedAt` | `Optional<Instant>` | yes | When the request was confirmed |
| `confirmedBy` | `Optional<String>` | yes | Confirmer identity |
| `confirmationNote` | `Optional<String>` | yes | Optional note from the confirmer |
| `cancelledAt` | `Optional<Instant>` | yes | When the request was cancelled |
| `cancelledBy` | `Optional<String>` | yes | Canceller identity |
| `cancelReason` | `Optional<String>` | yes | Cancel reason |
| `executedAt` | `Optional<String>` | yes | ISO-8601 execution time |
| `executionSummary` | `Optional<String>` | yes | Summary of what was executed |

Every nullable lifecycle field is `Optional<T>` because `Request` is the view row type (Lesson 6). `emptyState()` returns `Request.initial("")` with no `commandContext()` reference (Lesson 3).

## `RequestStatus` enum

`PENDING_CONFIRMATION`, `CONFIRMED`, `CANCELLED`, `EXECUTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `RequestPlanned` | `recordPlan` after PlannerAgent returns | sets `plannedAt`, `proposedActions`, status → `PENDING_CONFIRMATION` |
| `RequestConfirmed` | `confirm` command (human via API) | sets `confirmedAt`, `confirmedBy`, `confirmationNote`, status → `CONFIRMED` |
| `RequestCancelled` | `cancel` command (human via API) | sets `cancelledAt`, `cancelledBy`, `cancelReason`, status → `CANCELLED` |
| `RequestExecuted` | `recordExecution` after ExecutorAgent returns | sets `executedAt`, `executionSummary`, status → `EXECUTED` |

## Domain records

- `ActionPlan(List<String> actions)` — PlannerAgent result.
- `ConfirmationDecision(String confirmedBy, String note)` — confirm payload.
- `ExecutionResult(String summary, String executedAt)` — ExecutorAgent result.

## View row type

`RequestsView` rows are `Request`. One query: `getAllRequests` → `SELECT * AS requests FROM requests_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (ConfirmationTasks.java)

- `PLAN` — `Task.name(...).resultConformsTo(ActionPlan.class)`.
- `EXECUTE` — `Task.name(...).resultConformsTo(ExecutionResult.class)`.
