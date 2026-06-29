# Data model — durable-agent-baseline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WorkStep` | `stepId` | `String` | no | Stable id supplied by the submitter. |
| | `description` | `String` | no | What the step does, in plain English. |
| | `maxRetries` | `int` | no | How many retry attempts the agent may make before marking the step FAILED. |
| `WorkOrder` | `workOrderId` | `String` | no | UUID minted by `WorkOrderEndpoint`. |
| | `title` | `String` | no | User-supplied label. |
| | `steps` | `List<WorkStep>` | no | Submitted steps (1–N), in execution order. |
| | `submittedBy` | `String` | no | User or system identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `StepOutcome` | `stepId` | `String` | no | MUST equal a submitted `stepId`. |
| | `status` | `StepStatus` | no | Enum value. |
| | `output` | `String` | no | What the step produced or why it failed. Never empty. |
| | `retryCount` | `int` | no | 0 on first-attempt success; N on Nth retry success or failure. |
| | `startedAt` | `Instant` | no | When agent began processing this step. |
| | `finishedAt` | `Instant` | no | When this step resolved. |
| `WorkOrderResult` | `resolution` | `WorkOrderResolution` | no | Enum value. |
| | `summary` | `String` | no | 1–3 sentences from the agent. |
| | `outcomes` | `List<StepOutcome>` | no | One entry per submitted step, in order. |
| | `decidedAt` | `Instant` | no | When the agent returned the result. |
| `EvalScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence from `PerformanceEvaluator`. |
| | `evaluatedAt` | `Instant` | no | When `PerformanceEvaluator` finished. |
| `AlertEvent` | `kind` | `AlertKind` | no | Enum value. |
| | `stepId` | `String` | no | The step that triggered the alert. |
| | `detail` | `String` | no | Human-readable context (elapsed time, event count, etc.). |
| | `detectedAt` | `Instant` | no | When `RuntimeMonitor` detected the condition. |
| `WorkOrderRun` (entity state) | `workOrderId` | `String` | no | — |
| | `order` | `Optional<WorkOrder>` | yes | Populated after `WorkOrderInitiated`. |
| | `result` | `Optional<WorkOrderResult>` | yes | Populated after `WorkOrderCompleted`. |
| | `eval` | `Optional<EvalScore>` | yes | Populated after `PerformanceEvaluated`. |
| | `stepOutcomes` | `List<StepOutcome>` | no | Accumulates across `StepCompleted` events. |
| | `alerts` | `List<AlertEvent>` | no | Accumulates across `AlertRecorded` events. |
| | `status` | `WorkOrderStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `WorkOrderInitiated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `WorkOrderRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`StepStatus`: `PENDING`, `RUNNING`, `COMPLETED`, `FAILED`.
`WorkOrderResolution`: `COMPLETED`, `PARTIAL`, `FAILED`.
`AlertKind`: `STALL`, `RESOURCE_OVER_RUN`.
`WorkOrderStatus`: `INITIATED`, `RUNNING`, `STEP_COMPLETED`, `COMPLETED`, `EVALUATED`, `STALLED`, `FAILED`.

## Events (`WorkOrderEntity`)

| Event | Payload | Transition |
|---|---|---|
| `WorkOrderInitiated` | `order` | → INITIATED |
| `WorkOrderStarted` | — | → RUNNING |
| `StepStarted` | `stepId` | stays RUNNING (lateral) |
| `StepCompleted` | `stepId, outcome` | → STEP_COMPLETED (returns to RUNNING for next step) |
| `StepFailed` | `stepId, reason` | stays RUNNING (or STALLED if stall was active) |
| `WorkOrderCompleted` | `result` | → COMPLETED |
| `WorkOrderFailed` | `reason: String` | → FAILED (terminal) |
| `AlertRecorded` | `alert` | RUNNING → STALLED (for STALL kind); no transition for RESOURCE_OVER_RUN |
| `PerformanceEvaluated` | `eval` | → EVALUATED (terminal happy) |

`emptyState()` returns `WorkOrderRun.initial("")` with all `Optional` fields as `Optional.empty()`, `stepOutcomes = List.of()`, `alerts = List.of()`, and `status = INITIATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`WorkOrderRow` mirrors `WorkOrderRun` in full. All fields are included — there is no field equivalent to `rawDocument` in this baseline that must be withheld from the view. The view declares ONE query: `getAllWorkOrders: SELECT * AS workOrders FROM work_order_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`WorkOrderTasks.java`)

```java
public final class WorkOrderTasks {
  public static final Task<WorkOrderResult> PROCESS_WORK_ORDER = Task
      .name("Process work order")
      .description("Execute all steps in the work order and produce a WorkOrderResult")
      .resultConformsTo(WorkOrderResult.class);

  private WorkOrderTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
