# Data model — akka-llm-client-basic

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TurnRequest` | `prompt` | `String` | no | User-submitted prompt text. Blank not allowed — endpoint returns 400. |
| | `modelOverride` | `Optional<String>` | yes | Provider key to use for this turn; falls back to application default if absent. |
| `ConversationReply` | `text` | `String` | no | Model response. Guardrail asserts non-empty and length ≤ 8 000. |
| | `inputTokens` | `Optional<Integer>` | yes | Tokens in; provider-dependent. |
| | `outputTokens` | `Optional<Integer>` | yes | Tokens out; provider-dependent. |
| | `latencyMs` | `Optional<Long>` | yes | End-to-end agent-call latency in milliseconds; populated by endpoint. |
| | `generatedAt` | `Instant` | no | When the agent returned the reply. |
| `Turn` | `turnId` | `String` | no | UUID minted by `ConversationEndpoint`. |
| | `prompt` | `String` | no | The submitted prompt text. |
| | `reply` | `Optional<ConversationReply>` | yes | Populated after `TurnReplied`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `submittedAt` | `Instant` | no | When `TurnSubmitted` was emitted. |
| | `repliedAt` | `Optional<Instant>` | yes | Populated after `TurnReplied`. |
| `ConversationSession` | `sessionId` | `String` | no | UUID minted by `ConversationEndpoint`. |
| (entity state) | `turns` | `List<Turn>` | no | Ordered by `submittedAt`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` was emitted. |
| | `lastActivityAt` | `Optional<Instant>` | yes | Updated on every `TurnSubmitted`. |

Every nullable field on `ConversationSession` and `Turn` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `PENDING`, `REPLIED`, `FAILED`.
`SessionStatus`: `ACTIVE`, `CLOSED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `sessionId` | Session → ACTIVE |
| `TurnSubmitted` | `turnId`, `prompt`, `submittedAt` | Turn → PENDING |
| `TurnReplied` | `turnId`, `reply`, `repliedAt` | Turn → REPLIED |
| `TurnFailed` | `turnId`, `reason: String` | Turn → FAILED |

`emptyState()` returns `ConversationSession.initial("")` with `turns = new ArrayList<>()`, `status = ACTIVE`, and `lastActivityAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `ConversationSession` in full — turns are included in the row so the UI can render history from the view without re-querying the entity. The view is NOT filtered by session status (Lesson 2); the endpoint filters client-side.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM conversation_view`.

## Task definition (`ConversationTasks.java`)

```java
public final class ConversationTasks {
  public static final Task<ConversationReply> GENERATE_REPLY = Task
      .name("Generate reply")
      .description("Respond to the user prompt given prior conversation context")
      .resultConformsTo(ConversationReply.class);

  private ConversationTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Guardrail checks (`ReplyGuardrail.java`)

| Check id | Condition rejected | Structured error code |
|---|---|---|
| `EMPTY_TEXT` | `reply.text()` is blank | `empty-reply-text` |
| `OVER_LENGTH` | `reply.text().length() > 8000` | `reply-too-long` |
| `REFUSAL_EMPTY` | refusal metadata flag present AND `reply.text()` is blank | `structured-refusal-empty-body` |
