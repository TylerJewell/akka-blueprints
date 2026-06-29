# Data model — akka-bridge

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ToolDefinition` | `toolName` | `String` | no | Stable identifier for the tool (e.g. `web_search`). |
| | `description` | `String` | no | One-sentence description for the agent. |
| | `argSchema` | `String` | no | JSON Schema string for the tool's argument object. |
| `RunRequest` | `runId` | `String` | no | UUID minted by `RunEndpoint`. |
| | `taskDescription` | `String` | no | Plain-text task the agent must complete. |
| | `permittedTools` | `List<ToolDefinition>` | no | Tools the agent may call for this run. |
| | `toolBudget` | `int` | no | Maximum number of permitted tool calls. |
| | `submittedBy` | `String` | no | User or operator identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallRecord` | `callId` | `String` | no | UUID minted by `ToolCallGuardrail`. |
| | `toolName` | `String` | no | Name of the tool called. |
| | `argumentsJson` | `String` | no | Exact argument JSON sent by the agent. |
| | `guardrailVerdict` | `GuardrailVerdict` | no | Enum: `PERMITTED` or `BLOCKED`. |
| | `rejectionReason` | `Optional<String>` | yes | Non-null when `guardrailVerdict = BLOCKED`. |
| | `resultJson` | `Optional<String>` | yes | Non-null when the call was permitted and executed. |
| | `calledAt` | `Instant` | no | When the guardrail intercepted the call. |
| | `completedAt` | `Optional<Instant>` | yes | When `AgentFrameworkAdapter` wrote the result. |
| `RunResult` | `answer` | `String` | no | Concise final answer from the agent. |
| | `toolCallsUsed` | `int` | no | Count of `PERMITTED` calls made. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `Run` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<RunRequest>` | yes | Populated after `RunAccepted`. |
| | `toolCalls` | `List<ToolCallRecord>` | no | Empty list in initial state; appended per call. |
| | `result` | `Optional<RunResult>` | yes | Populated after `RunCompleted`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunAccepted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Run` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`GuardrailVerdict`: `PERMITTED`, `BLOCKED`.
`RunStatus`: `ACCEPTED`, `RUNNING`, `COMPLETED`, `BLOCKED`, `FAILED`.

## Events (`RunEntity`)

| Event | Payload | Transition / Effect |
|---|---|---|
| `RunAccepted` | `request` | → ACCEPTED |
| `RunStarted` | — | → RUNNING |
| `ToolCallRequested` | `callId, toolName, argumentsJson` | appends `ToolCallRecord` (no verdict yet) |
| `ToolCallPermitted` | `callId` | sets `guardrailVerdict = PERMITTED` on the matching record |
| `ToolCallBlocked` | `callId, rejectionReason` | sets `guardrailVerdict = BLOCKED` on the matching record |
| `ToolResultRecorded` | `callId, resultJson` | sets `resultJson` and `completedAt` on the matching record |
| `RunCompleted` | `result` | → COMPLETED (terminal happy) |
| `RunBlocked` | `reason: String` | → BLOCKED (terminal — budget or irrecoverable tool block) |
| `RunFailed` | `reason: String` | → FAILED (terminal — agent error, timeout) |

`emptyState()` returns `Run.initial("")` with `toolCalls = Collections.emptyList()`, all `Optional` fields as `Optional.empty()`, and `status = ACCEPTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `Run`. The UI fetches full tool-call details on demand via `GET /api/runs/{id}`. The view's row type includes the full `toolCalls` list so the UI can render the live tool-call log from SSE events.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RunTasks.java`)

```java
public final class RunTasks {
  public static final Task<RunResult> RUN_TASK = Task
      .name("Run agent task")
      .description("Execute the given task using the permitted tools and return a RunResult")
      .resultConformsTo(RunResult.class);

  private RunTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
