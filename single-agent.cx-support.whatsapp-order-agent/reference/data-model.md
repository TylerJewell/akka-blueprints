# Data model — whatsapp-order-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Product` | `sku` | `String` | no | Unique product identifier. |
| | `name` | `String` | no | Human-readable product name. |
| | `description` | `String` | no | Short product description. |
| | `stockQuantity` | `int` | no | Current units available. |
| | `unitPrice` | `double` | no | Price per unit. |
| | `currency` | `String` | no | ISO 4217 code (e.g., `"USD"`). |
| `OrderLineItem` | `sku` | `String` | no | SKU being ordered. |
| | `productName` | `String` | no | Snapshot of product name at order time. |
| | `quantity` | `int` | no | Units requested (must be > 0). |
| | `unitPrice` | `double` | no | Price snapshot at order time. |
| | `lineTotal` | `double` | no | `quantity × unitPrice`. |
| `OrderRequest` | `orderId` | `String` | no | UUID minted by the agent tool call. |
| | `sessionId` | `String` | no | Owning session. |
| | `customerId` | `String` | no | Customer placing the order. |
| | `items` | `List<OrderLineItem>` | no | One entry per SKU. |
| | `orderTotal` | `double` | no | Sum of all `lineTotal` values. |
| | `currency` | `String` | no | ISO 4217 code. |
| | `deliveryAddress` | `String` | no | Delivery address (may be redacted in view). |
| | `requestedAt` | `Instant` | no | When the tool call was made. |
| `AgentReply` | `replyText` | `String` | no | Natural-language message to the customer. |
| | `toolCallsSummary` | `List<ToolCall>` | no | One entry per tool invoked or attempted. |
| | `hitlRequired` | `boolean` | no | True when `pendingOrderTotal > HITL_THRESHOLD`. |
| | `pendingOrderTotal` | `double` | no | 0.0 if no order created; order total otherwise. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `ToolCall` | `toolName` | `String` | no | e.g., `"create-order"`. |
| | `arguments` | `String` | no | JSON string of arguments passed. |
| | `outcome` | `String` | no | `"allowed"` or `"blocked"`. |
| | `blockReason` | `String` | yes | `null` when allowed; guardrail reason when blocked. |
| `TurnContext` | `turnId` | `String` | no | UUID minted by the endpoint. |
| | `customerMessage` | `String` | no | Sanitized customer message. |
| | `agentReply` | `String` | no | Agent's reply text. |
| | `piiCategoriesRedacted` | `List<String>` | no | e.g., `["phone","address"]`. Empty list if none. |
| `TurnEval` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `evaluatedAt` | `Instant` | no | When `TurnScorer` finished. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `customerId` | `String` | no | — |
| | `status` | `SessionStatus` | no | See enum. |
| | `turns` | `List<TurnContext>` | no | Ordered; may be empty at session start. |
| | `pendingOrder` | `Optional<OrderRequest>` | yes | Set when `hitlRequired`; cleared after approval/rejection. |
| | `lastTurnEval` | `Optional<TurnEval>` | yes | Most recent turn's eval score. |
| | `createdAt` | `Instant` | no | When `SessionStarted` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `Order` (entity state) | `orderId` | `String` | no | — |
| | `sessionId` | `String` | no | — |
| | `status` | `OrderStatus` | no | See enum. |
| | `request` | `Optional<OrderRequest>` | yes | Populated after `OrderDrafted`. |
| | `createdAt` | `Instant` | no | When `OrderDrafted` emitted. |
| | `confirmedAt` | `Optional<Instant>` | yes | When `OrderConfirmed` emitted. |
| | `cancelledAt` | `Optional<Instant>` | yes | When `OrderCancelled` emitted. |

Every nullable field on `Session` and `Order` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`SessionStatus`: `IDLE`, `ACTIVE`, `AWAITING_APPROVAL`, `COMPLETING`, `CLOSED`, `FAILED`.
`OrderStatus`: `DRAFT`, `CONFIRMED`, `SHIPPED`, `CANCELLED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionStarted` | `customerId` | → IDLE |
| `TurnReceived` | `customerMessage`, `turnId` | → ACTIVE |
| `TurnCompleted` | `agentReply` (AgentReply), `turnId` | stays ACTIVE |
| `ConversationSanitized` | `sanitizedTurn` (TurnContext), `turnId` | stays ACTIVE (turn list updated) |
| `ApprovalRequested` | `orderId`, `orderTotal` | → AWAITING_APPROVAL |
| `ApprovalGranted` | `orderId` | → COMPLETING |
| `ApprovalRejected` | `orderId`, `reason` | → FAILED |
| `SessionCompleted` | — | → CLOSED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

## Events (`OrderEntity`)

| Event | Payload | Transition |
|---|---|---|
| `OrderDrafted` | `request` (OrderRequest) | → DRAFT |
| `OrderConfirmed` | — | → CONFIRMED |
| `OrderShipped` | — | → SHIPPED |
| `OrderCancelled` | `reason: String` | → CANCELLED (terminal) |
| `OrderFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` on both entities returns `<Entity>.initial("")` with all `Optional` fields as `Optional.empty()` and `turns` as `List.of()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session` but the `turns` list contains only the sanitized `TurnContext` entries (no raw pre-sanitization text). `OrderRequest.deliveryAddress` in the view row is always the redacted form. The raw delivery address and raw customer messages are accessible only via the entity's event log, not through the view.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`OrderTasks.java`)

```java
public final class OrderTasks {
  public static final Task<AgentReply> ORDER_TASKS = Task
      .name("Handle customer order request")
      .description("Read the customer message and conversation history, call appropriate product/order tools, and return an AgentReply")
      .resultConformsTo(AgentReply.class);

  private OrderTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
