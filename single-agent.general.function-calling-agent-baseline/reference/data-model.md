# Data model — function-calling-agent-baseline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ToolDefinition` | `toolName` | `String` | no | Registry name used to invoke the tool. Must match a key in `InProcessToolExecutor`. |
| | `description` | `String` | no | Human-readable description passed to the agent in the task instructions. |
| | `parameterTypes` | `Map<String, String>` | no | Parameter name → type hint (`"number"` or `"string"`). Used by `ToolCallGuardrail`. |
| `RunRequest` | `runId` | `String` | no | UUID minted by `AgentRunEndpoint`. |
| | `query` | `String` | no | The user's natural-language question. |
| | `enabledTools` | `List<ToolDefinition>` | no | Tools available to the agent for this run (0–N). |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallRecord` | `iteration` | `int` | no | Which LLM iteration produced this call (1-indexed). |
| | `toolName` | `String` | no | Name of the tool the agent called. |
| | `arguments` | `Map<String, Object>` | no | Parameter name → value as passed by the agent. |
| | `result` | `String` | no | Serialised tool output. Empty string if the call was blocked. |
| | `blocked` | `boolean` | no | `true` if `ToolCallGuardrail` rejected this invocation. |
| | `calledAt` | `Instant` | no | When the tool invocation was attempted. |
| `GuardrailSummary` | `toolCallBlocked` | `boolean` | no | Whether any tool call was blocked during this run. |
| | `answerBlocked` | `boolean` | no | Whether any final answer was blocked during this run. |
| | `toolCallBlockCount` | `int` | no | Total number of tool-call rejections. |
| | `answerBlockCount` | `int` | no | Total number of answer rejections. |
| `AgentAnswer` | `answer` | `String` | no | The agent's final answer to the user's query. |
| | `toolCallTrace` | `List<ToolCallRecord>` | no | Ordered list of all tool calls made (including blocked ones). |
| | `totalIterations` | `int` | no | How many LLM iterations the agent used. |
| | `guardrailSummary` | `GuardrailSummary` | no | Aggregated guardrail activity for this run. |
| | `answeredAt` | `Instant` | no | When the agent returned the final answer. |
| `AgentRun` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<RunRequest>` | yes | Populated after `RunSubmitted`. |
| | `answer` | `Optional<AgentAnswer>` | yes | Populated after `AnswerRecorded`. |
| | `status` | `AgentRunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `AgentRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AgentRunStatus`: `SUBMITTED`, `RUNNING`, `ANSWER_RECORDED`, `FAILED`.

## Events (`AgentRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunSubmitted` | `request: RunRequest` | → SUBMITTED |
| `RunStarted` | — | → RUNNING |
| `AnswerRecorded` | `answer: AgentAnswer` | → ANSWER_RECORDED (terminal happy) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `AgentRun.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`AgentRunRow` mirrors `AgentRun`. The UI fetches the full run on demand via `GET /api/runs/{id}` when the user selects a card.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM agent_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`AgentRunTasks.java`)

```java
public final class AgentRunTasks {
  public static final Task<AgentAnswer> RUN_QUERY = Task
      .name("Run query")
      .description("Process the user's query using available tools and return an AgentAnswer")
      .resultConformsTo(AgentAnswer.class);

  private AgentRunTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool registry (`InProcessToolExecutor.java`)

Three in-process tools registered with the Akka function-calling mechanism:

| Tool name | Parameters | Return | Notes |
|---|---|---|---|
| `calculator` | `expression: String` | `String` (numeric result) | Evaluates integer/decimal arithmetic with `+`, `-`, `*`, `/`. Returns `"Error: ..."` on parse failure. |
| `dictionary` | `word: String` | `String` (definition or `"Not found."`) | Looks up from a ~30-word in-process map. Case-insensitive. |
| `date-formatter` | `isoDate: String`, `format: String` | `String` (formatted date) | Accepts `format` values: `"short"` (e.g., `"Jun 28, 2026"`), `"long"` (e.g., `"June 28, 2026"`), `"iso"` (identity). |
