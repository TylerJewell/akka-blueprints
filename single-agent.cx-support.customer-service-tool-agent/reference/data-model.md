# Data model — customer-service-tool-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `CustomerMessage` | `messageId` | `String` | no | UUID minted by `SupportEndpoint`. |
| | `customerId` | `String` | no | Customer identifier from the request. |
| | `text` | `String` | no | Raw message text as submitted. |
| | `sentAt` | `Instant` | no | When the endpoint received the message. |
| `ToolCall` | `toolName` | `String` | no | Name of the called tool (e.g., `lookupOrder`). |
| | `arguments` | `Map<String, String>` | no | Key-value arguments passed to the tool. |
| | `rawResult` | `String` | no | JSON-serialised tool return value, or `"BLOCKED"`. |
| | `outcome` | `ToolCallOutcome` | no | `ALLOWED` or `BLOCKED`. |
| | `blockReason` | `Optional<String>` | yes | Populated when `outcome == BLOCKED`. |
| `AgentReply` | `replyText` | `String` | no | Customer-facing reply text (max 1000 chars). |
| | `toolCalls` | `List<ToolCall>` | no | All tool calls made during this turn (may be empty list if mock). |
| | `outcome` | `ReplyOutcome` | no | `SENT` or `BLOCKED_AND_RETRIED`. |
| | `repliedAt` | `Instant` | no | When the agent returned the reply. |
| `ScreenedPayload` | `redactedReplyText` | `String` | no | PII-redacted form of `AgentReply.replyText`. |
| | `piiCategoriesFound` | `List<String>` | no | e.g. `["email","payment-card-number","phone"]`. |
| `ConversationTurn` | `message` | `CustomerMessage` | no | The customer's message for this turn. |
| | `reply` | `Optional<AgentReply>` | yes | Populated once the agent returns. |
| `Conversation` (entity state) | `conversationId` | `String` | no | UUID minted on open. |
| | `customerId` | `String` | no | Customer identifier. |
| | `turns` | `List<ConversationTurn>` | no | Ordered list of turns; grows with each message. |
| | `lastScreened` | `Optional<ScreenedPayload>` | yes | Populated after `ReplyScreened` for the most recent turn. |
| | `status` | `ConversationStatus` | no | See enum. |
| | `openedAt` | `Instant` | no | When `ConversationOpened` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Conversation` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ToolCallOutcome`: `ALLOWED`, `BLOCKED`.
`ReplyOutcome`: `SENT`, `BLOCKED_AND_RETRIED`.
`ConversationStatus`: `OPEN`, `ACTIVE`, `REPLIED`, `ESCALATED`, `RESOLVED`, `FAILED`.

## Tool result types

| Record | Field | Type | Meaning |
|---|---|---|---|
| `OrderRecord` | `orderId` | `String` | Seeded order id. |
| | `customerId` | `String` | Owning customer. |
| | `productSku` | `String` | Product ordered. |
| | `status` | `String` | e.g. `SHIPPED`, `DELIVERED`, `PROCESSING`. |
| | `carrier` | `Optional<String>` | Shipping carrier name. |
| | `trackingNumber` | `Optional<String>` | Carrier tracking ref. |
| | `totalAmount` | `double` | Order total in USD. |
| | `estimatedDelivery` | `Optional<Instant>` | ETA. |
| `AccountRecord` | `customerId` | `String` | — |
| | `name` | `String` | Full name. |
| | `email` | `String` | Contact email. |
| | `phone` | `Optional<String>` | Contact phone. |
| `UpdateResult` | `success` | `boolean` | Whether the update was applied. |
| | `message` | `String` | Confirmation or error description. |
| `InventoryRecord` | `productSku` | `String` | — |
| | `productName` | `String` | Human-readable name. |
| | `stockLevel` | `int` | Current units available. |
| | `available` | `boolean` | `stockLevel > 0`. |
| `EscalationTicket` | `ticketId` | `String` | e.g. `ESC-9910`. |
| | `customerId` | `String` | — |
| | `reason` | `String` | Operator-readable reason. |
| | `amount` | `double` | Disputed or requested amount. |
| | `status` | `String` | `OPEN` (always at creation). |
| | `createdAt` | `Instant` | — |

## Events (`ConversationEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ConversationOpened` | `customerId: String` | → OPEN |
| `MessageReceived` | `message: CustomerMessage` | stays in current status (OPEN or REPLIED) |
| `AgentProcessingStarted` | — | → ACTIVE |
| `AgentReplied` | `reply: AgentReply` | → REPLIED |
| `ReplyScreened` | `screened: ScreenedPayload` | stays REPLIED |
| `ConversationEscalated` | `escalationTicketId: String` | → ESCALATED (terminal) |
| `ConversationResolved` | — | → RESOLVED (terminal) |
| `ConversationFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Conversation.initial("")` with `turns = List.of()`, all `Optional` fields as `Optional.empty()`, and `status = OPEN`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ConversationRow` mirrors `Conversation`. The UI renders the full turn list including tool-calls summaries. The raw `reply.replyText` is included in the row (for the right-pane thread); the screened form is separately available as `lastScreened.redactedReplyText`. The UI always displays `lastScreened.redactedReplyText` for the agent reply text.

The view declares ONE query: `getAllConversations: SELECT * AS conversations FROM conversation_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SupportTasks.java`)

```java
public final class SupportTasks {
  public static final Task<AgentReply> HANDLE_SUPPORT_QUERY = Task
      .name("Handle support query")
      .description("Read the conversation history, call the appropriate tools, and return an AgentReply")
      .resultConformsTo(AgentReply.class);

  private SupportTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
