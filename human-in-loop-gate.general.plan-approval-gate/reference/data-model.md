# Data model

Every record, event, enum, and view row the generated system defines.

## `Plan` (PlanEntity state + PlansView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Plan id; equals the workflow id and entity id |
| `goal` | `Optional<String>` | yes | The submitted goal |
| `status` | `PlanStatus` | no | Lifecycle status |
| `plannedAt` | `Optional<Instant>` | yes | When the plan was first recorded |
| `steps` | `Optional<List<String>>` | yes | Ordered list of execution steps |
| `rationale` | `Optional<String>` | yes | Explanation of why the steps achieve the goal |
| `editedAt` | `Optional<Instant>` | yes | When the plan was last edited |
| `editedBy` | `Optional<String>` | yes | Identity of the person who edited the plan |
| `approvedAt` | `Optional<Instant>` | yes | When the plan was approved |
| `approvedBy` | `Optional<String>` | yes | Identity of the approver |
| `approverNote` | `Optional<String>` | yes | Approver's note |
| `cancelledAt` | `Optional<Instant>` | yes | When the plan was cancelled |
| `cancelledBy` | `Optional<String>` | yes | Identity of the person who cancelled |
| `cancelReason` | `Optional<String>` | yes | Cancellation reason |
| `executedAt` | `Optional<String>` | yes | ISO-8601 execution completion time |
| `outcome` | `Optional<String>` | yes | Description of what was accomplished |

Every nullable lifecycle field is `Optional<T>` because `Plan` is the view row type (Lesson 6). `emptyState()` returns `Plan.initial("")` with no `commandContext()` reference (Lesson 3).

## `PlanStatus` enum

`PLANNED`, `APPROVED`, `CANCELLED`, `EXECUTED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `PlanCreated` | `recordPlan` after PlanningAgent returns | sets `plannedAt`, `steps`, `rationale`, status → `PLANNED` |
| `PlanEdited` | `editPlan` command (human via API) | sets `editedAt`, `editedBy`, updates `steps`; status stays `PLANNED` |
| `PlanApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `PlanCancelled` | `cancel` command (human via API) | sets `cancelledAt`, `cancelledBy`, `cancelReason`, status → `CANCELLED` |
| `PlanExecuted` | `recordExecution` after ExecutionAgent returns | sets `executedAt`, `outcome`, status → `EXECUTED` |

## Domain records

- `AgentPlan(List<String> steps, String rationale)` — PlanningAgent result.
- `PlanEdit(String editedBy, List<String> revisedSteps)` — edit payload.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `ExecutionResult(String outcome, String completedAt)` — ExecutionAgent result.

## View row type

`PlansView` rows are `Plan`. One query: `getAllPlans` → `SELECT * AS plans FROM plans_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (PlanTasks.java)

- `PLAN` — `Task.name(...).resultConformsTo(AgentPlan.class)`.
- `EXECUTE` — `Task.name(...).resultConformsTo(ExecutionResult.class)`.
