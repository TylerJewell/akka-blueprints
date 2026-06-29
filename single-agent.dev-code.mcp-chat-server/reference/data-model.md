# Data model — mcp-chat-server

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `UserMessage` | `turnId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `conversationId` | `String` | no | Parent conversation. |
| | `text` | `String` | no | The message the user sent. |
| | `sentBy` | `String` | no | User identifier. |
| | `sentAt` | `Instant` | no | When the endpoint received the request. |
| `ToolCall` | `toolName` | `String` | no | Name of the MCP tool invoked or attempted. |
| | `serverUrl` | `String` | no | MCP server URL the call was directed at. |
| | `inputSummary` | `String` | no | Short description of the input; not the raw payload. |
| | `outcome` | `ToolCallOutcome` | no | `ALLOWED` or `BLOCKED`. |
| | `calledAt` | `Instant` | no | When the invocation was attempted. |
| `ChatReply` | `replyText` | `String` | no | The reply shown to the user. |
| | `toolCallsAudited` | `List<ToolCall>` | no | Every tool call attempted (allowed or blocked). |
| | `iterationsUsed` | `int` | no | How many agent iterations this turn consumed. |
| | `repliedAt` | `Instant` | no | When the agent returned the reply. |
| `Turn` | `turnId` | `String` | no | UUID minted by `ChatEndpoint`. |
| | `message` | `UserMessage` | no | The user's message for this turn. |
| | `reply` | `Optional<ChatReply>` | yes | Populated after `ReplyRecorded`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `failureReason` | `Optional<String>` | yes | Populated after `TurnFailed`. |
| | `startedAt` | `Instant` | no | When `MessageReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `title` | `String` | no | User-supplied label. |
| | `createdBy` | `String` | no | User identifier. |
| | `turns` | `List<Turn>` | no | Ordered oldest-first; may be empty. |
| | `status` | `ConversationStatus` | no | `ACTIVE` or `CLOSED`. |
| | `createdAt` | `Instant` | no | When `ConversationCreated` emitted. |
| | `lastActiveAt` | `Optional<Instant>` | yes | Updated after each `ReplyRecorded`. |

Every nullable field on `Turn` and `Conversation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ToolCallOutcome`: `ALLOWED`, `BLOCKED`.
`TurnStatus`: `RECEIVED`, `RUNNING`, `REPLIED`, `FAILED`.
`ConversationStatus`: `ACTIVE`, `CLOSED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationCreated` | `title`, `createdBy`, `createdAt` | → conversation ACTIVE |
| `MessageReceived` | `message` | → new Turn appended, status RECEIVED |
| `AgentRunStarted` | `turnId` | → Turn status RUNNING |
| `ReplyRecorded` | `turnId`, `reply` | → Turn status REPLIED (terminal happy) |
| `TurnFailed` | `turnId`, `reason: String` | → Turn status FAILED (terminal) |
| `ConversationClosed` | — | → conversation CLOSED |

`emptyState()` returns `Conversation.initial("")` with empty `turns` list, `status = ACTIVE`, `lastActiveAt = Optional.empty()`, and no `commandContext()` reference (Lesson 3). `Optional<T>` fields use `Optional.empty()` in initial state and `Optional.of(...)` inside event-appliers.

## View row

`ConversationRow` mirrors `Conversation` in full, including the embedded `List<TurnRow>`. `TurnRow` mirrors `Turn`. The view serves both list queries and the single-conversation detail without a second fetch.

The view declares TWO queries:

```
getAllConversations: SELECT * AS conversations FROM conversation_view
getConversation: SELECT * FROM conversation_view WHERE conversationId = :conversationId
```

No `WHERE status = :status` filter on the list query — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChatTasks.java`)

```java
public final class ChatTasks {
  public static final Task<ChatReply> CHAT_TURN = Task
      .name("Chat turn")
      .description("Read the user message and conversation history, call MCP tools as needed, and return a ChatReply")
      .resultConformsTo(ChatReply.class);

  private ChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
