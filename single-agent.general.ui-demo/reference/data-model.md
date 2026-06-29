# Data model — ui-demo

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Turn` | `turnId` | `String` | no | UUID minted by `CopilotEndpoint`. |
| | `prompt` | `String` | no | User-supplied prompt text. |
| | `response` | `Optional<CopilotResponse>` | yes | Populated after `TurnCompleted`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `submittedAt` | `Instant` | no | When the endpoint received the POST. |
| | `completedAt` | `Optional<Instant>` | yes | When `TurnCompleted` emitted. |
| `CopilotResponse` | `answer` | `String` | no | Answer text with inline `[n]` citation markers. Non-blank. |
| | `citations` | `List<Citation>` | no | One entry per marker used in answer. |
| | `suggestedFollowUp` | `String` | no | One question the user could ask next. |
| | `tokenCount` | `int` | no | Approximate token count of the `answer` field. |
| | `generatedAt` | `Instant` | no | When the agent returned. |
| `Citation` | `index` | `int` | no | 1-based; matches a `[n]` marker in `answer`. Positive. |
| | `source` | `String` | no | Non-empty label or reference. |
| | `snippet` | `String` | no | Verbatim or near-verbatim supporting passage. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `title` | `String` | no | User-supplied label. |
| | `turns` | `List<Turn>` | no | All turns in submission order. Empty list on creation. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `lastActiveAt` | `Optional<Instant>` | yes | Updated on each `TurnCompleted`. |

Every nullable field on `Turn` and `Session` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `SUBMITTED`, `STREAMING`, `TURN_COMPLETE`, `TURN_FAILED`.
`SessionStatus`: `ACTIVE`, `IDLE`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `title` | → ACTIVE (initial) |
| `TurnSubmitted` | `turn` (status=SUBMITTED) | IDLE → ACTIVE |
| `StreamingStarted` | `turnId` | ACTIVE → ACTIVE (turn status: SUBMITTED → STREAMING) |
| `TurnCompleted` | `turnId`, `response` | ACTIVE → IDLE (turn status: STREAMING → TURN_COMPLETE) |
| `TurnFailed` | `turnId`, `reason: String` | ACTIVE → FAILED (turn status: STREAMING → TURN_FAILED) |

`emptyState()` returns `Session.initial("")` with an empty `turns` list, `status = ACTIVE`, and `lastActiveAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors the fields needed by the session list UI: `sessionId`, `title`, `status`, `turnCount` (derived from `turns.size()`), `createdAt`, `lastActiveAt`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`CopilotTasks.java`)

```java
public final class CopilotTasks {
  public static final Task<CopilotResponse> ANSWER_PROMPT = Task
      .name("Answer prompt")
      .description("Read the conversation context and produce a CopilotResponse with answer, citations, and a suggested follow-up question")
      .resultConformsTo(CopilotResponse.class);

  private CopilotTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## AG-UI SSE envelope types

| Envelope type | Fields | When emitted |
|---|---|---|
| `TOKEN_CHUNK` | `text: String` | Each token chunk as the agent produces output. |
| `CITATION_ADDED` | `index: int`, `source: String`, `snippet: String` | When a citation entry is identified in the structured output. |
| `RESPONSE_COMPLETE` | `turnId: String`, `response: CopilotResponse` | Once after `TurnCompleted` lands in the entity. Closes the stream. |
