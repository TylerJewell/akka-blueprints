# Data model — gemini-fullstack

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChatMessage` | `messageId` | `String` | no | UUID minted by `ChatEndpoint` on each POST. |
| | `role` | `MessageRole` | no | `USER` or `AGENT`. |
| | `content` | `String` | no | Message text. |
| | `tokenCount` | `Optional<Integer>` | yes | Present on `AGENT` messages; absent on `USER` messages. |
| | `createdAt` | `Instant` | no | When the message was added to the entity. |
| `SendMessageRequest` | `conversationId` | `String` | no | The target conversation. |
| | `userMessage` | `String` | no | Raw user text. |
| | `submittedBy` | `String` | no | User identifier. |
| `AgentReply` | `content` | `String` | no | Agent reply text. |
| | `tokenCount` | `int` | no | Approximate token count for the reply. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `Conversation` (entity state) | `conversationId` | `String` | no | — |
| | `title` | `String` | no | User-supplied label. |
| | `messages` | `List<ChatMessage>` | no | All turns, chronological. Empty list in initial state. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ConversationCreated` emitted. |
| | `lastActivityAt` | `Optional<Instant>` | yes | Updated on every `UserMessageAdded` and `AgentReplyRecorded`. |

Every nullable field on `Conversation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`MessageRole`: `USER`, `AGENT`.

`ConversationStatus`: `CREATED`, `ACTIVE`, `AWAITING_REPLY`, `REPLY_RECORDED`, `FAILED`.

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationCreated` | `title: String, submittedBy: String` | → CREATED |
| `UserMessageAdded` | `message: ChatMessage` | → ACTIVE (from CREATED or REPLY_RECORDED) |
| `ReplyGenerationStarted` | `messageId: String` | → AWAITING_REPLY |
| `AgentReplyRecorded` | `messageId: String, reply: AgentReply` | → REPLY_RECORDED |
| `ConversationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Conversation.initial("")` with an empty `messages` list, `status = CREATED`, and `lastActivityAt = Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

The `messages` list is append-only. `AgentReplyRecorded` finds the matching user message by `messageId` and appends the agent reply as a new `ChatMessage` with `role = AGENT`.

## View row

`ConversationRow` mirrors `Conversation`. The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ChatTasks.java`)

```java
public final class ChatTasks {
  public static final Task<AgentReply> GENERATE_REPLY = Task
      .name("Generate reply")
      .description("Read the conversation context and produce an AgentReply.")
      .resultConformsTo(AgentReply.class);

  private ChatTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Utility class (`ConversationContextBuilder.java`)

Pure stateless utility. Inputs: `List<ChatMessage> messages`, `int contextWindowSize`. Output: `byte[]` (UTF-8 JSON array of the last `contextWindowSize` messages, each as `{ "role": "USER"|"AGENT", "content": "..." }`). No external dependencies. Used by `ConversationWorkflow.generateStep` to build the task attachment.
