# Data model — entity-workflow-chat

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `UserMessage` | `messageId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `rawText` | `String` | no | Pre-sanitization message body. Audit-only. |
| | `sentAt` | `Instant` | no | When the endpoint received the request. |
| `SanitizedMessage` | `messageId` | `String` | no | Matches the parent `UserMessage`. |
| | `redactedText` | `String` | no | PII redacted; this is what the agent context receives. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","phone","payment-card-number","address","person-name"]`. |
| | `sanitizedAt` | `Instant` | no | When `MessageSanitizer` finished. |
| `AgentReply` | `messageId` | `String` | no | Matches the turn's `messageId`. |
| | `replyText` | `String` | no | The agent's response text (1–8 sentences). |
| | `suggestClose` | `boolean` | no | True only if the issue appears resolved. |
| | `repliedAt` | `Instant` | no | When `ConversationAgent` returned. |
| `Turn` | `turnId` | `String` | no | Matches the `messageId` of the originating user message. |
| | `userMessage` | `SanitizedMessage` | no | Populated after `MessageSanitized`. |
| | `agentReply` | `Optional<AgentReply>` | yes | Populated after `AgentReplied`. |
| | `status` | `TurnStatus` | no | See enum. |
| `ConversationSummary` | `summaryText` | `String` | no | Bullet-list of topics and resolutions from compacted turns. |
| | `turnsCompacted` | `int` | no | Number of turns collapsed into this summary. |
| | `compactedAt` | `Instant` | no | When `ContextCompactor` ran. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `topic` | `String` | no | User-supplied label. |
| | `userId` | `String` | no | User identifier from the creation request. |
| | `turns` | `List<Turn>` | no | All turns, oldest first. Empty list at creation. |
| | `summaries` | `List<ConversationSummary>` | no | All compaction summaries. Empty list if no compaction. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversationStarted` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Conversation` and `Turn` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `WAITING_SANITIZE`, `READY`, `AGENT_THINKING`, `COMPLETED`, `FAILED`.
`ConversationStatus`: `OPEN`, `AGENT_THINKING`, `COMPACTING`, `CLOSED`, `FAILED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationStarted` | `topic, userId` | → OPEN |
| `MessageReceived` | `userMessage` | (no status change — triggers Consumer) |
| `MessageSanitized` | `messageId, sanitized` | OPEN → AGENT_THINKING |
| `AgentReplied` | `reply` | AGENT_THINKING → OPEN |
| `HistoryCompacted` | `summary` | COMPACTING → OPEN |
| `ConversationClosed` | `closedAt` | OPEN → CLOSED (terminal) |
| `ConversationFailed` | `reason: String` | AGENT_THINKING → FAILED (terminal) |

`emptyState()` returns `Conversation.initial("")` with `turns` and `summaries` as empty lists, `closedAt = Optional.empty()`, and `status = OPEN`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `Conversation` except that each `Turn`'s `userMessage` field exposes the `SanitizedMessage` (not the raw `UserMessage`). The audit log on the entity keeps the raw text via `MessageReceived` events; the view only surfaces the sanitized form.

The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChatTasks.java`)

```java
public final class ChatTasks {
  public static final Task<AgentReply> REPLY_TO_TURN = Task
      .name("Reply to turn")
      .description("Read the conversation context attachment and produce an AgentReply for the latest user message")
      .resultConformsTo(AgentReply.class);

  private ChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Context package (attachment schema)

The `agentStep` serialises this structure to JSON and attaches it as `context.json`:

```java
record ContextPackage(
    String summary,               // null if no compaction has occurred
    List<SanitizedTurn> recentTurns
) {}

record SanitizedTurn(
    String turnId,
    String userMessage,           // redactedText from SanitizedMessage
    String agentReply             // null for the current unanswered turn
) {}
```

The last entry in `recentTurns` is always the current turn (`agentReply == null`). This is the message `ConversationAgent` must reply to.
