# Data model — activity-interrupt-cancellation

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ActivityTask` | `taskId` | `String` | no | Unique id, generated at submission. |
| | `name` | `String` | no | Short human-readable task name. |
| | `description` | `String` | no | Free-text description of the work to perform. |
| | `kind` | `TaskKind` | no | One of `INFRA_PROVISION`, `REPORT_GENERATE`, `DATA_PIPELINE`, `MAINTENANCE`. |
| | `estimatedSteps` | `int` | no | Expected number of steps; used by the agent for progress tracking. |
| | `submittedAt` | `Instant` | no | When the task entered the queue. |
| `StepResult` | `stepIndex` | `int` | no | 1-based step counter. |
| | `summary` | `String` | no | One-sentence description of the step outcome. |
| | `terminal` | `boolean` | no | `true` if no further steps should run. |
| | `partialArtifact` | `Optional<String>` | yes | Resource ID or output fragment, when produced. |
| | `completedAt` | `Instant` | no | When this step finished. |
| `CleanupPlan` | `steps` | `List<String>` | no | Ordered remediation steps. |
| | `reason` | `String` | no | Why cleanup is needed. |
| | `rollbackNote` | `Optional<String>` | yes | Rollback-specific note, when applicable. |
| | `planCreatedAt` | `Instant` | no | When `CleanupAgent` produced this plan. |
| `CancellationRequest` | `requestedBy` | `String` | no | Operator identifier or `"reaper"` for timeout-triggered cancellations. |
| | `reason` | `String` | no | Human-readable reason. |
| | `requestedAt` | `Instant` | no | When the request was received. |
| `ActivityRecord` (entity state) | `taskId` | `String` | no | — |
| | `task` | `ActivityTask` | no | Captured once at queue time. |
| | `stepLog` | `List<StepResult>` | no | All steps that completed (empty until RUNNING). |
| | `cleanupPlan` | `Optional<CleanupPlan>` | yes | Populated after `CleanupRecorded`. |
| | `cancellationRequest` | `Optional<CancellationRequest>` | yes | Populated after `CancellationRequested`. |
| | `status` | `ActivityStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TaskQueued` was emitted. |
| | `startedAt` | `Optional<Instant>` | yes | When `TaskStarted` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal state timestamp. |

## Enums

`TaskKind`: `INFRA_PROVISION`, `REPORT_GENERATE`, `DATA_PIPELINE`, `MAINTENANCE`.

`ActivityStatus`: `QUEUED`, `RUNNING`, `CANCELLING`, `CANCELLED`, `COMPLETED`, `FAILED`.

## Events (`ActivityEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskQueued` | `task` | → QUEUED |
| `TaskStarted` | — | → RUNNING |
| `StepCompleted` | `stepResult` | (no status change; appends to stepLog) |
| `CancellationRequested` | `cancellationRequest` | → CANCELLING |
| `CleanupRecorded` | `cleanupPlan` | (no status change; sets cleanupPlan) |
| `TaskCancelled` | — | → CANCELLED (terminal) |
| `TaskCompleted` | — | → COMPLETED (terminal) |
| `TaskFailed` | `reason: String` | → FAILED (terminal) |

## Events (`TaskQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `task` (the full `ActivityTask`; immutable audit record) |

## View row

`ActivityRow` mirrors `ActivityRecord`. The `task.description` field is omitted from the view row (kept in the entity for detail fetches). All optional fields use `Optional<T>` in the view row record; `/akka:implement` must not use raw nullable references.
