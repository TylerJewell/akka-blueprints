# Data model — durable-workflow-recovery

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CheckpointRecord` | `checkpointId` | `String` | no | Stable id for this checkpoint within the execution. |
| | `phase` | `String` | no | Name of the workflow phase (e.g., "extract", "transform"). |
| | `elapsedMs` | `long` | no | Wall-clock time the phase took to reach this checkpoint. |
| | `outcome` | `CheckpointOutcome` | no | Enum: COMPLETED, FAILED, SKIPPED, PENDING. |
| | `recordedAt` | `Instant` | no | When the checkpoint event was received. |
| `ExecutionRegistration` | `executionId` | `String` | no | UUID minted by `RecoveryEndpoint`. |
| | `workflowId` | `String` | no | Upstream workflow identifier. |
| | `workflowType` | `String` | no | Type tag used to look up baseline latency and expected checkpoint count. |
| | `ownerTeam` | `String` | no | Team responsible for this execution. |
| | `registeredAt` | `Instant` | no | When the endpoint received the registration. |
| `CheckpointSnapshot` | `checkpoints` | `List<CheckpointRecord>` | no | All checkpoint records at stall time, ordered by `recordedAt`. |
| | `totalExpected` | `int` | no | Expected total checkpoint count for this workflow type. |
| | `completedCount` | `int` | no | Count of checkpoints with `outcome == COMPLETED`. |
| | `failedCount` | `int` | no | Count of checkpoints with `outcome == FAILED`. |
| | `stalledFor` | `Duration` | no | Time elapsed since the last checkpoint event. |
| `CheckpointStatus` | `checkpointId` | `String` | no | MUST equal a `checkpointId` in the snapshot. |
| | `phase` | `String` | no | Copied from the snapshot record. |
| | `outcome` | `CheckpointOutcome` | no | Enum value. |
| | `recommendedAction` | `String` | no | Actionable verb-phrase for the operator. |
| `RecoveryDecision` | `verdict` | `RecoveryVerdict` | no | Enum: RESUME, ABORT, ESCALATE. |
| | `rationale` | `String` | no | 2–4 sentences explaining the verdict. |
| | `checkpointStatuses` | `List<CheckpointStatus>` | no | One entry per checkpoint in the snapshot. |
| | `decidedAt` | `Instant` | no | When the agent returned. |
| `HealthScore` | `score` | `int` | no | 1–5. |
| | `diagnosis` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `HealthEvaluator` finished. |
| `Execution` (entity state) | `executionId` | `String` | no | — |
| | `registration` | `Optional<ExecutionRegistration>` | yes | Populated after `ExecutionRegistered`. |
| | `snapshot` | `Optional<CheckpointSnapshot>` | yes | Populated after `StallDetected`. |
| | `decision` | `Optional<RecoveryDecision>` | yes | Populated after `DecisionRecorded`. |
| | `health` | `Optional<HealthScore>` | yes | Populated after `HealthScored`. |
| | `status` | `ExecutionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ExecutionRegistered` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Execution` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`CheckpointOutcome`: `COMPLETED`, `FAILED`, `SKIPPED`, `PENDING`.
`RecoveryVerdict`: `RESUME`, `ABORT`, `ESCALATE`.
`ExecutionStatus`: `REGISTERED`, `RUNNING`, `STALLED`, `ANALYZING`, `DECISION_RECORDED`, `HEALTH_SCORED`, `ABORTED`, `FAILED`.

## Events (`ExecutionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ExecutionRegistered` | `registration` | → REGISTERED |
| `CheckpointRecorded` | `checkpoint` | → RUNNING (first); stays RUNNING (subsequent) |
| `StallDetected` | `stalledFor: Duration` | → STALLED |
| `AnalysisStarted` | — | → ANALYZING |
| `DecisionRecorded` | `decision` | → DECISION_RECORDED (RESUME/ESCALATE) or ABORTED (ABORT) |
| `HealthScored` | `health` | → HEALTH_SCORED (terminal happy) |
| `ExecutionAborted` | — | → ABORTED (terminal, ABORT path) |
| `ExecutionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Execution.initial("")` with all `Optional` fields as `Optional.empty()` and `status = REGISTERED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ExecutionRow` mirrors `Execution`. The UI fetches the full execution detail on demand via `GET /api/executions/{id}` and reads the nested `snapshot.checkpoints` list from the JSON for the checkpoint timeline.

The view declares ONE query: `getAllExecutions: SELECT * AS executions FROM execution_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RecoveryTasks.java`)

```java
public final class RecoveryTasks {
  public static final Task<RecoveryDecision> ANALYZE_EXECUTION = Task
      .name("Analyze execution")
      .description("Read the checkpoint snapshot and produce a RecoveryDecision per checkpoint")
      .resultConformsTo(RecoveryDecision.class);

  private RecoveryTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## ResponseValidator return type

```java
record ValidationResult(boolean valid, String reason) {
  static ValidationResult ok() { return new ValidationResult(true, ""); }
  static ValidationResult fail(String reason) { return new ValidationResult(false, reason); }
}
```

Used inside `RecoveryWorkflow.analyzeStep` before calling `ExecutionEntity.recordDecision`. A `valid = false` result causes the step to transition to the error step.
