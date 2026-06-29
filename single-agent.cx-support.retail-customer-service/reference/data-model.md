# Data model — retail-customer-service

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Product` | `productId` | `String` | no | Stable catalog identifier. |
| | `name` | `String` | no | Display name. |
| | `category` | `String` | no | e.g. `"succulent"`, `"fern"`, `"herb"`, `"tool"`. |
| | `description` | `String` | no | 1–3 sentence product description. |
| | `priceUsd` | `double` | no | Catalog price in USD. |
| | `inStock` | `boolean` | no | Current availability. |
| | `careInstructions` | `String` | no | Watering, light, soil guidance. |
| `OrderLineItem` | `productId` | `String` | no | References a `Product`. |
| | `productName` | `String` | no | Denormalized at order time. |
| | `quantity` | `int` | no | Units ordered. |
| | `unitPriceUsd` | `double` | no | Price at order time. |
| `OrderModification` | `modificationType` | `String` | no | `"cancel"`, `"update-address"`, `"update-quantity"`. |
| | `field` | `String` | no | Which field changed. |
| | `previousValue` | `String` | no | Value before the change. |
| | `newValue` | `String` | no | Value after the change. |
| | `appliedAt` | `Instant` | no | When the change was applied. |
| `Order` | `orderId` | `String` | no | UUID minted at order placement. |
| | `customerId` | `String` | no | Owner customer. |
| | `lineItems` | `List<OrderLineItem>` | no | 1–N items. |
| | `shippingAddress` | `String` | no | Current delivery address. |
| | `status` | `OrderStatus` | no | See enum. |
| | `placedAt` | `Instant` | no | When the order was placed. |
| | `shippedAt` | `Optional<Instant>` | yes | Set when status transitions to SHIPPED. |
| | `deliveredAt` | `Optional<Instant>` | yes | Set when status transitions to DELIVERED. |
| | `modifications` | `List<OrderModification>` | no | Audit trail of changes; empty list initially. |
| `ConversationTurn` | `turnId` | `String` | no | UUID minted per turn. |
| | `customerMessage` | `String` | no | The customer's input text. |
| | `reply` | `Optional<AgentReply>` | yes | Populated after TurnReplied. |
| | `orderContext` | `Optional<String>` | yes | orderId if the turn was order-scoped. |
| | `status` | `TurnStatus` | no | See enum. |
| | `startedAt` | `Instant` | no | When the turn began. |
| | `completedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| `AgentReply` | `message` | `String` | no | Customer-facing text. |
| | `orderChange` | `Optional<OrderChangeRequest>` | yes | Present if a modification was requested. |
| | `model` | `String` | no | Model identifier. |
| | `repliedAt` | `Instant` | no | When the agent returned. |
| `OrderChangeRequest` | `orderId` | `String` | no | Target order. |
| | `changeType` | `String` | no | `"cancel"`, `"update-address"`, `"update-quantity"`. |
| | `field` | `String` | no | Affected field. |
| | `newValue` | `String` | no | Requested new value. |
| `Session` | `sessionId` | `String` | no | UUID minted at session open. |
| | `customerId` | `String` | no | Customer owning the session. |
| | `turns` | `List<ConversationTurn>` | no | Ordered list of turns; empty list initially. |
| | `status` | `SessionStatus` | no | See enum. |
| | `openedAt` | `Instant` | no | When the session was opened. |
| | `closedAt` | `Optional<Instant>` | yes | Set when session closes. |

Every nullable lifecycle field uses `Optional<T>`. The view table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`OrderStatus`: `PENDING`, `PROCESSING`, `SHIPPED`, `DELIVERED`, `CANCELLED`.
`TurnStatus`: `PENDING`, `REPLIED`, `GUARDRAIL_BLOCKED`, `FAILED`.
`SessionStatus`: `OPEN`, `ACTIVE`, `CLOSED`.

## Events — `SessionEntity`

| Event | Payload | Transition |
|---|---|---|
| `SessionOpened` | `sessionId, customerId` | → OPEN |
| `TurnStarted` | `turnId, customerMessage, orderContext?` | → ACTIVE (stays ACTIVE on subsequent turns) |
| `TurnReplied` | `turnId, agentReply` | ACTIVE (turn status → REPLIED) |
| `TurnBlockedByGuardrail` | `turnId, rejectionCode` | ACTIVE (turn status → GUARDRAIL_BLOCKED) |
| `TurnFailed` | `turnId, reason` | ACTIVE (turn status → FAILED) |
| `SessionClosed` | — | → CLOSED (terminal) |

## Events — `OrderEntity`

| Event | Payload | Transition |
|---|---|---|
| `OrderPlaced` | `orderId, customerId, lineItems, shippingAddress` | → PENDING |
| `OrderProcessing` | — | → PROCESSING |
| `OrderShipped` | `shippedAt` | → SHIPPED |
| `OrderDelivered` | `deliveredAt` | → DELIVERED (terminal) |
| `OrderCancelled` | `cancelledAt` | → CANCELLED (terminal) |
| `OrderAddressUpdated` | `previousAddress, newAddress` | (stays in current status) |
| `OrderQuantityUpdated` | `productId, previousQty, newQty` | (stays in current status) |

`emptyState()` returns `Session.initial("")` and `Order.initial("")` with all `Optional` fields as `Optional.empty()` and empty lists. `emptyState()` never references `commandContext()` (Lesson 3).

## View rows

### `SessionRow` (SessionView)

Mirrors `Session` summary. Excludes the full `turns` list — callers fetch the full session via `GET /api/sessions/{id}`. Includes `lastTurnStatus`, `lastReplyPreview` (first 120 chars of last reply), and `turnCount` for the list UI.

View declares two queries:
- `getAllSessions: SELECT * AS sessions FROM session_view` — all sessions, no WHERE clause (Lesson 2; caller filters by status client-side).

### `OrderRow` (OrderView)

Mirrors `Order`. View declares:
- `getOrder: SELECT * AS orders FROM order_view WHERE orderId = :orderId`
- `getAllOrders: SELECT * AS orders FROM order_view`

## Task definition (`CustomerServiceTasks.java`)

```java
public final class CustomerServiceTasks {
  public static final Task<AgentReply> HANDLE_CUSTOMER_TURN = Task
      .name("Handle customer turn")
      .description("Read the customer message and context attachment, answer the question or process the order change, and return an AgentReply")
      .resultConformsTo(AgentReply.class);

  private CustomerServiceTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
