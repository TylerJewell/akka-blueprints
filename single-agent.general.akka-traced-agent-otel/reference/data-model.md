# Data model — traced-agent-otel

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SpanRecord` | `spanId` | `String` | no | Short unique id for this span (8 hex chars). |
| | `parentSpanId` | `Optional<String>` | yes | `null` for the root span; parent's `spanId` for child spans. |
| | `traceId` | `String` | no | Shared by all spans in one `AgentResponse`. |
| | `operationName` | `String` | no | e.g. `"llm.completion"`, `"tool.web_search"`, `"memory.read"`. |
| | `kind` | `SpanKind` | no | Enum value. |
| | `startTime` | `Instant` | no | When the operation began. |
| | `endTime` | `Instant` | no | When the operation ended; must be ≥ `startTime`. |
| | `attributes` | `Map<String,String>` | no | e.g. `{"model":"claude-sonnet-4-6","token.count":"218"}`. |
| | `status` | `SpanStatus` | no | Enum value. |
| | `errorMessage` | `Optional<String>` | yes | Non-null only when `status == ERROR`. |
| `SpanSummary` | `traceId` | `String` | no | Matches the `AgentResponse.spans[0].traceId`. |
| | `totalSpans` | `int` | no | Count of all `SpanRecord` entries. |
| | `llmTurnCount` | `int` | no | Count of spans with `kind == LLM_CALL`. |
| | `toolCallCount` | `int` | no | Count of spans with `kind == TOOL_CALL`. |
| | `errorCount` | `int` | no | Count of spans with `status == ERROR`. |
| | `wallClockMillis` | `long` | no | Millis between earliest `startTime` and latest `endTime`. |
| | `spans` | `List<SpanRecord>` | no | Full span list (mirrors `AgentResponse.spans`). |
| | `collectedAt` | `Instant` | no | When `SpanCollector` finished. |
| `PromptRequest` | `conversationId` | `String` | no | UUID minted by `ConversationEndpoint`. |
| | `promptText` | `String` | no | User-supplied prompt body. |
| | `toolModeEnabled` | `boolean` | no | When true, agent includes tool-call spans. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `AgentResponse` | `answerText` | `String` | no | The agent's answer to the prompt. |
| | `spans` | `List<SpanRecord>` | no | Spans emitted during the agent run. |
| | `respondedAt` | `Instant` | no | When the agent returned. |
| `PerformanceMonitorResult` | `qualityScore` | `int` | no | 1–5. |
| | `p50Millis` | `long` | no | Median LLM_CALL span duration in ms. |
| | `p95Millis` | `long` | no | 95th-percentile LLM_CALL span duration in ms. |
| | `toolSuccessRate` | `double` | no | OK tool spans / total tool spans; 1.0 if no tool spans. |
| | `rationale` | `String` | no | One sentence explaining the score. |
| | `monitoredAt` | `Instant` | no | When `PerformanceMonitor` finished. |
| `FlushResult` | `spansExported` | `int` | no | Count of spans successfully sent to the exporter. |
| | `spansDropped` | `int` | no | Count of spans not sent (exporter error or limit). |
| | `exporterHealthy` | `boolean` | no | True if the exporter returned no error. |
| | `zipkinTraceUrl` | `Optional<String>` | yes | URL to view the trace; absent if exporter was unhealthy. |
| | `flushedAt` | `Instant` | no | When the flush step finished. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `request` | `Optional<PromptRequest>` | yes | Populated after `PromptSubmitted`. |
| | `agentResponse` | `Optional<AgentResponse>` | yes | Populated after `AgentRunCompleted`. |
| | `spanSummary` | `Optional<SpanSummary>` | yes | Populated after `SpanSummaryAttached`. |
| | `flushResult` | `Optional<FlushResult>` | yes | Populated after `TraceFlushed`. |
| | `monitorResult` | `Optional<PerformanceMonitorResult>` | yes | Populated after `PerformanceMonitorScored`. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `PromptSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Conversation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SpanKind`: `LLM_CALL`, `TOOL_CALL`, `MEMORY_READ`, `MEMORY_WRITE`, `WORKFLOW_STEP`.
`SpanStatus`: `OK`, `ERROR`, `UNSET`.
`ConversationStatus`: `SUBMITTED`, `RUNNING`, `COMPLETED`, `FLUSHED`, `MONITORED`, `FAILED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PromptSubmitted` | `request` | → SUBMITTED |
| `AgentRunStarted` | — | → RUNNING |
| `AgentRunCompleted` | `agentResponse` | → COMPLETED |
| `SpanSummaryAttached` | `spanSummary` | (no status change; enriches COMPLETED row) |
| `TraceFlushed` | `flushResult` | → FLUSHED |
| `PerformanceMonitorScored` | `monitorResult` | → MONITORED (terminal happy) |
| `ConversationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Conversation.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `Conversation` minus `agentResponse.spans` raw list (the list can be large; the view holds only the `SpanSummary` aggregates). The full span list is available via `GET /api/conversations/{id}` which returns the complete entity state.

The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ConversationTasks.java`)

```java
public final class ConversationTasks {
  public static final Task<AgentResponse> ANSWER_PROMPT = Task
      .name("Answer prompt")
      .description("Process the user's prompt, emit OpenTelemetry spans for every LLM turn "
          + "and tool call, and return an AgentResponse with the answer text and the "
          + "collected span records")
      .resultConformsTo(AgentResponse.class);

  private ConversationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
