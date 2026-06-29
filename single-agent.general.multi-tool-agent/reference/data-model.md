# Data model — multi-tool-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `SessionRequest` | `sessionId` | `String` | no | UUID minted by `SessionEndpoint`. |
| | `requestText` | `String` | no | User's natural-language question. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCallRecord` | `callId` | `String` | no | Unique id per call attempt, minted by the agent. |
| | `toolName` | `String` | no | `"WeatherTool"`, `"CurrencyTool"`, or `"UnitTool"`. |
| | `inputArgs` | `Map<String, String>` | no | Tool-specific input arguments as string pairs. |
| | `result` | `String` | no | JSON result from the tool, or guardrail error text if rejected. |
| | `guardRejectionReason` | `Optional<String>` | yes | Null on success; the rejection message on guardrail reject. |
| | `calledAt` | `Instant` | no | When the tool call was attempted. |
| `WeatherResult` | `city` | `String` | no | Normalised city name. |
| | `condition` | `String` | no | e.g. `"Partly cloudy"`. |
| | `tempCelsius` | `double` | no | Current temperature in Celsius. |
| | `humidity` | `int` | no | Relative humidity percentage (0–100). |
| `CurrencyResult` | `fromCode` | `String` | no | ISO-4217 source currency. |
| | `toCode` | `String` | no | ISO-4217 target currency. |
| | `amount` | `double` | no | Input amount. |
| | `converted` | `double` | no | Converted amount. |
| | `rate` | `double` | no | Exchange rate applied. |
| `UnitResult` | `fromUnit` | `String` | no | Source unit label. |
| | `toUnit` | `String` | no | Target unit label. |
| | `inputValue` | `double` | no | Input value. |
| | `outputValue` | `double` | no | Converted value. |
| `ToolResponse` | `answer` | `String` | no | 1–4 sentence combined answer. |
| | `toolCalls` | `List<ToolCallRecord>` | no | All calls made, including rejected attempts. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `request` | `Optional<SessionRequest>` | yes | Populated after `SessionSubmitted`. |
| | `toolCalls` | `List<ToolCallRecord>` | no | Starts empty; appended by `ToolCallRecorded`. |
| | `response` | `Optional<ToolResponse>` | yes | Populated after `SessionAnswered`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Session` is `Optional<T>`. The `toolCalls` field starts as `List.of()` and is rebuilt via `List.copyOf` in the `ToolCallRecorded` applier (Lesson 6).

## Enums

`SessionStatus`: `SUBMITTED`, `DISPATCHING`, `ANSWERED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionSubmitted` | `request` | → SUBMITTED |
| `DispatchStarted` | — | → DISPATCHING |
| `ToolCallRecorded` | `record: ToolCallRecord` | (no status change; appends to toolCalls list) |
| `SessionAnswered` | `response: ToolResponse` | → ANSWERED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Session.initial("")` with `request = Optional.empty()`, `toolCalls = List.of()`, `response = Optional.empty()`, `finishedAt = Optional.empty()`, `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session` in full (there is no audit-only field to strip, unlike the docreview blueprint where `rawDocument` is omitted from the view). The UI fetches the session detail via `GET /api/sessions/{id}`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SessionTasks.java`)

```java
public final class SessionTasks {
  public static final Task<ToolResponse> ANSWER_REQUEST = Task
      .name("Answer request")
      .description("Use available tools to answer the user's natural-language question and return a combined ToolResponse")
      .resultConformsTo(ToolResponse.class);

  private SessionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
