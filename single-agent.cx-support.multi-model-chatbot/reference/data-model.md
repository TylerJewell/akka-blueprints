# Data model — multi-model-chatbot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MessageTurn` | `turnId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `userText` | `String` | no | Raw user message. Audit-only. |
| | `sanitizedText` | `Optional<String>` | yes | PII-redacted user text. Populated after `MessageSanitized`. |
| | `piiCategoriesFound` | `List<String>` | no | Empty list if none found. e.g. `["email","phone","account-id"]`. |
| | `reply` | `Optional<ChatReply>` | yes | Populated after `ReplyGenerated`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `receivedAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `repliedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `ChatReply` | `replyText` | `String` | no | Full reply content returned by the agent. |
| | `providerName` | `String` | no | e.g. `"anthropic"`, `"openai"`, `"googleai-gemini"`. |
| | `modelId` | `String` | no | e.g. `"claude-sonnet-4-6"`, `"gpt-4o"`. |
| | `inputTokens` | `int` | no | Estimated input token count. |
| | `outputTokens` | `int` | no | Estimated output token count. |
| | `generatedAt` | `Instant` | no | When the agent returned the reply. |
| `ConversationSession` | `sessionId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `userId` | `String` | no | User identifier from the session request. |
| | `displayName` | `String` | no | User-supplied label for the session. |
| | `activeProvider` | `String` | no | Current provider for new turns. |
| | `turns` | `List<MessageTurn>` | no | All turns in chronological order. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionStarted` emitted. |
| | `lastActivityAt` | `Optional<Instant>` | yes | Timestamp of the most recent turn event. |

Every nullable lifecycle field is `Optional<T>`. The view table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `RECEIVED`, `SANITIZED`, `REPLIED`, `FAILED`.
`SessionStatus`: `ACTIVE`, `CLOSED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionStarted` | `sessionId, userId, displayName, activeProvider` | Session created (ACTIVE). |
| `MessageReceived` | `turnId, userText, receivedAt` | New turn → RECEIVED. |
| `MessageSanitized` | `turnId, sanitizedText, piiCategoriesFound` | Turn → SANITIZED. |
| `ReplyGenerated` | `turnId, reply` | Turn → REPLIED (terminal happy). |
| `TurnFailed` | `turnId, reason` | Turn → FAILED (terminal). |
| `ProviderChanged` | `newProvider` | Session `activeProvider` updated; no turn transition. |
| `SessionClosed` | — | Session → CLOSED. |

`emptyState()` returns `ConversationSession.initial("")` with empty `turns` list, `status = ACTIVE`, and `lastActivityAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `ConversationSession` with full `turns` list. The view exposes the same data the chat thread UI needs — including `sanitizedText` on each turn. The raw `userText` on each turn is also present (it is not stripped from the view, because the view is not user-facing — it is consumed by `ChatEndpoint`, which serves it only to the UI. The UI renders `sanitizedText` in the chat thread; `userText` is available in the JSON for supervisors).

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChatTasks.java`)

```java
public final class ChatTasks {
  public static final Task<ChatReply> GENERATE_REPLY = Task
      .name("Generate reply")
      .description("Read the conversation history and produce a ChatReply for the latest user message")
      .resultConformsTo(ChatReply.class);

  private ChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
