# Data model — async-agent-endpoint

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskPrompt` | `taskId` | `String` | no | UUID minted by `RunEndpoint` (same as `runId`). |
| | `promptText` | `String` | no | Natural-language task description from the user. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `GeneratedCode` | `code` | `String` | no | Python source that was executed. |
| | `language` | `String` | no | Always `"python"` in this blueprint. |
| `RunResult` | `stdout` | `String` | no | Captured stdout from the execution (may be empty). |
| | `stderr` | `String` | no | Captured stderr (empty on success). |
| | `outputValue` | `String` | no | Final expression value as a string (empty if none). |
| | `guardrailHitOnAnyIteration` | `boolean` | no | True if at least one prior iteration was rejected. |
| | `code` | `GeneratedCode` | no | The code that ran on the successful iteration. |
| | `completedAt` | `Instant` | no | When the agent returned the result. |
| `RunFailure` | `kind` | `FailureKind` | no | Enum value. |
| | `message` | `String` | no | Human-readable failure description. |
| | `failedAt` | `Instant` | no | When the workflow error step recorded the failure. |
| `AgentRun` (entity state) | `runId` | `String` | no | — |
| | `prompt` | `Optional<TaskPrompt>` | yes | Populated after `RunSubmitted`. |
| | `result` | `Optional<RunResult>` | yes | Populated after `RunCompleted`. |
| | `failure` | `Optional<RunFailure>` | yes | Populated after `RunFailed`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `AgentRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`RunStatus`: `SUBMITTED`, `RUNNING`, `COMPLETED`, `FAILED`.
`FailureKind`: `GUARDRAIL_EXHAUSTED`, `EXECUTION_ERROR`, `TIMEOUT`.

## Events (`AgentRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunSubmitted` | `prompt` | → SUBMITTED |
| `RunStarted` | — | → RUNNING |
| `RunCompleted` | `result` | → COMPLETED (terminal happy) |
| `RunFailed` | `failure` | → FAILED (terminal) |

`emptyState()` returns `AgentRun.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `AgentRun` fields. The view declares ONE query: `getAllRuns: SELECT * AS runs FROM run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RunTasks.java`)

```java
public final class RunTasks {
  public static final Task<RunResult> RUN_TASK = Task
      .name("Run task")
      .description("Generate Python code to satisfy the task prompt, execute it, and return a RunResult")
      .resultConformsTo(RunResult.class);

  private RunTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
