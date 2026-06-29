# Data model — react-chatbot

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `UserMessage` | `turnId` | `String` | no | UUID minted by `ChatEndpoint` at send time. |
| | `content` | `String` | no | The user's message text. |
| | `sentBy` | `String` | no | User identifier from the request body. |
| | `sentAt` | `Instant` | no | When the endpoint received the send request. |
| `ToolCall` | `toolName` | `String` | no | One of: `search_knowledge_base`, `get_order_status`, `get_faq_answer`. |
| | `inputSummary` | `String` | no | Brief description of the tool's input (e.g., `orderId = ORD-003`). |
| | `resultSummary` | `String` | no | Brief description of what the tool returned. |
| `ChatReply` | `content` | `String` | no | The agent's final plain-prose answer. |
| | `toolCallTrace` | `List<ToolCall>` | no | Ordered list of tool calls made before the final answer. May be empty if no tools were needed. |
| | `repliedAt` | `Instant` | no | When the agent returned the reply. |
| `Turn` | `turnId` | `String` | no | Matches `UserMessage.turnId`. |
| | `message` | `UserMessage` | no | The user's message for this turn. |
| | `reply` | `Optional<ChatReply>` | yes | Populated after `ReplyRecorded`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `failureReason` | `Optional<String>` | yes | Populated after `TurnFailed`. |
| | `startedAt` | `Instant` | no | When `TurnStarted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `Conversation` (entity state) | `conversationId` | `String` | no | UUID minted by `ChatEndpoint` at creation time. |
| | `title` | `String` | no | User-supplied label. |
| | `createdBy` | `String` | no | User identifier from the request body. |
| | `turns` | `List<Turn>` | no | Ordered list of all turns. Starts empty. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversationStarted` emitted. |
| | `lastActiveAt` | `Optional<Instant>` | yes | Updated on each `TurnStarted` or `ReplyRecorded`. |

Every nullable field on `Turn` and `Conversation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`TurnStatus`: `PROCESSING`, `COMPLETED`, `FAILED`.
`ConversationStatus`: `ACTIVE`, `CLOSED`.

## Events (`ConversationEntity`)

| Event | Payload | Effect |
|---|---|---|
| `ConversationStarted` | `title: String, createdBy: String` | Initialises entity; status = ACTIVE. |
| `TurnStarted` | `message: UserMessage` | Appends a new `Turn` in PROCESSING to `turns`. Updates `lastActiveAt`. |
| `ReplyRecorded` | `turnId: String, reply: ChatReply` | Finds the turn by `turnId`; sets `reply`, `status = COMPLETED`, `finishedAt`. Updates `lastActiveAt`. |
| `TurnFailed` | `turnId: String, reason: String` | Finds the turn by `turnId`; sets `failureReason`, `status = FAILED`, `finishedAt`. |
| `ConversationClosed` | — | Sets `status = CLOSED`. |

`emptyState()` returns `Conversation.initial("")` with `turns = Collections.emptyList()` and `status = ACTIVE`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `Conversation` including all `Turn` entries. The view holds the full conversation history so the UI can render the thread without additional fetches.

The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChatTasks.java`)

```java
public final class ChatTasks {
  public static final Task<ChatReply> CHAT_TURN = Task
      .name("Chat turn")
      .description("Process the user message, call any needed tools, and return a ChatReply")
      .resultConformsTo(ChatReply.class);

  private ChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Tool registry stubs

| Tool | Input | Output record |
|---|---|---|
| `search_knowledge_base` | `query: String` | `List<KnowledgeArticle{title, snippet, url}>` |
| `get_order_status` | `orderId: String` | `OrderStatus{orderId, status, estimatedDelivery}` or not-found sentinel |
| `get_faq_answer` | `topic: String` | `FaqAnswer{topic, answer, relatedTopics: List<String>}` or not-found sentinel |

Stubs are loaded from `src/main/resources/tools/<tool-name>.json` at startup. The tool result is always a Java record or a not-found sentinel — the agent never receives a null.
