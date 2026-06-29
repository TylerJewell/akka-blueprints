# Data model — react-loop-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ToolSpec` | `toolName` | `String` | no | Unique name used in ACTION steps to identify the tool. |
| | `description` | `String` | no | One-sentence description the agent reads to decide which tool to call. |
| | `paramSchema` | `String` | no | JSON string describing the expected params object for this tool. |
| `RunRequest` | `runId` | `String` | no | UUID minted by `RunEndpoint`. |
| | `query` | `String` | no | The user's question or task. |
| | `availableTools` | `List<ToolSpec>` | no | Tools the agent may call for this run. |
| | `deniedTools` | `List<String>` | no | Tool names the guardrail will block regardless of availability. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `Step` | `stepIndex` | `int` | no | Zero-based position in the trace. |
| | `kind` | `StepKind` | no | Enum value: THOUGHT, ACTION, OBSERVATION, ANSWER. |
| | `content` | `String` | no | Thought text / action JSON / observation text / final answer. |
| | `toolName` | `String` | yes | Non-null for ACTION and OBSERVATION steps. |
| | `blocked` | `boolean` | no | True when `ToolCallGuardrail` blocked this ACTION. |
| | `recordedAt` | `Instant` | no | When this step was committed to the entity. |
| `ReActResult` | `finalAnswer` | `String` | no | The agent's final answer text. |
| | `steps` | `List<Step>` | no | Full step trace at the moment the result was recorded. |
| | `outcome` | `Outcome` | no | Enum value: ANSWERED or EXHAUSTED. |
| | `decidedAt` | `Instant` | no | When the final answer step was recorded. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `evaluatedAt` | `Instant` | no | When `ChainEvaluator` finished. |
| `Run` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<RunRequest>` | yes | Populated after `RunSubmitted`. |
| | `steps` | `List<Step>` | no | Grows with each `StepRecorded` / `ActionBlocked` event. Never null. |
| | `result` | `Optional<ReActResult>` | yes | Populated after `RunCompleted` or `RunExhausted`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `EvaluationScored`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Run` is `Optional<T>`. `steps` is `List.of()` initially and accumulated immutably in each event-applier (Lesson 6).

## Enums

`StepKind`: `THOUGHT`, `ACTION`, `OBSERVATION`, `ANSWER`.
`Outcome`: `ANSWERED`, `EXHAUSTED`.
`RunStatus`: `PENDING`, `RUNNING`, `COMPLETED`, `EXHAUSTED`, `FAILED`.

## Events (`RunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunSubmitted` | `request` | → PENDING |
| `RunStarted` | — | → RUNNING |
| `StepRecorded` | `step` | stays RUNNING; appends step |
| `ActionBlocked` | `step`, `reason: String` | stays RUNNING; appends blocked step |
| `RunCompleted` | `result` | → COMPLETED |
| `RunExhausted` | `result` | → EXHAUSTED |
| `EvaluationScored` | `eval` | stays COMPLETED or EXHAUSTED; appends eval |
| `RunFailed` | `reason: String` | → FAILED |

`emptyState()` returns `Run.initial("")` with `steps = List.of()`, all `Optional` fields as `Optional.empty()`, and `status = PENDING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `Run` in full (including the `steps` list) so the UI can render the step trace without a separate fetch. The view declares ONE query: `getAllRuns: SELECT * AS runs FROM run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`RunTasks.java`)

```java
public final class RunTasks {
  public static final Task<ReActResult> RUN_QUERY = Task
      .name("Run query")
      .description("Execute the ReAct loop for the given query using the available tools and return a ReActResult")
      .resultConformsTo(ReActResult.class);

  private RunTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool stub interface

All three in-process tool stubs implement the same informal interface:

```java
interface ToolStub {
  String name();         // matches ToolSpec.toolName
  String paramSchema();  // JSON string, same as ToolSpec.paramSchema
  String execute(String paramsJson);  // returns observation text; never throws
}
```

`ToolDispatcher` holds a `Map<String, ToolStub>` and resolves by `toolName`. Unknown names return `"Unknown tool: <name>"` as the observation (no exception).
