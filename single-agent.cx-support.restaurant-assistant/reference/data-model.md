# Data model — restaurant-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `MenuItem` | `itemId` | `String` | no | Stable catalog id (e.g., `"main-001"`). |
| | `name` | `String` | no | Display name. |
| | `category` | `String` | no | `STARTER`, `MAIN`, `DESSERT`, `DRINK`. |
| | `description` | `String` | no | Short menu description. |
| | `priceGBP` | `double` | no | Current price. |
| | `vegetarian` | `boolean` | no | Dietary flag. |
| | `vegan` | `boolean` | no | Dietary flag (implies vegetarian). |
| | `glutenFriendly` | `boolean` | no | Dietary flag. |
| | `allergens` | `List<String>` | no | e.g., `["dairy","nuts"]`. |
| `CustomerMessage` | `messageId` | `String` | no | UUID minted by endpoint. |
| | `role` | `String` | no | `CUSTOMER` or `ASSISTANT`. |
| | `text` | `String` | no | Message body. |
| | `sentAt` | `Instant` | no | When the message was recorded. |
| `ReservationRequest` | `date` | `String` | no | ISO-8601 date e.g. `"2026-07-04"`. |
| | `time` | `String` | no | HH:mm e.g. `"19:00"`. |
| | `partySize` | `int` | no | Number of guests (1–20). |
| | `guestName` | `String` | no | Lead guest name. |
| | `contactPhone` | `String` | no | Contact number. |
| `Reservation` | `reservationId` | `String` | no | UUID minted at confirmation. |
| | `details` | `ReservationRequest` | no | The validated request. |
| | `confirmedAt` | `Instant` | no | When `ReservationConfirmed` event was emitted. |
| `OrderLineItem` | `itemId` | `String` | no | Must match a catalog item id. |
| | `itemName` | `String` | no | Denormalised from catalog at order time. |
| | `quantity` | `int` | no | ≥ 1. |
| | `unitPriceGBP` | `double` | no | Denormalised from catalog at order time. |
| `Order` | `orderId` | `String` | no | UUID minted at commitment. |
| | `lines` | `List<OrderLineItem>` | no | ≥ 1 item. |
| | `type` | `OrderType` | no | Enum value. |
| | `totalGBP` | `double` | no | Sum of `quantity * unitPriceGBP` across lines. |
| | `placedAt` | `Instant` | no | When `OrderCommitted` event was emitted. |
| `AssistantResponse` | `replyText` | `String` | no | What the customer sees. |
| | `toolName` | `Optional<String>` | yes | `"makeReservation"` / `"submitOrder"` / empty. |
| | `reservationArg` | `Optional<ReservationRequest>` | yes | Present when `toolName = "makeReservation"`. |
| | `orderArg` | `Optional<List<OrderLineItem>>` | yes | Present when `toolName = "submitOrder"`. |
| | `respondedAt` | `Instant` | no | When the agent returned. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `messages` | `List<CustomerMessage>` | no | Full history; may be empty. |
| | `reservation` | `Optional<Reservation>` | yes | Populated after `ReservationConfirmed`. |
| | `order` | `Optional<Order>` | yes | Populated after `OrderCommitted`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `openedAt` | `Instant` | no | When `SessionOpened` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Session` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`OrderType`: `DINE_IN`, `TAKEAWAY`.
`SessionStatus`: `OPEN`, `ACTIVE`, `RESERVATION_HELD`, `ORDER_PLACED`, `CLOSED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionOpened` | `sessionId`, `openedAt` | → OPEN |
| `MessageAdded` | `message` (role=CUSTOMER) | OPEN → ACTIVE; ACTIVE stays ACTIVE |
| `AgentReplied` | `message` (role=ASSISTANT) | stays in current status |
| `ReservationConfirmed` | `reservation` | → RESERVATION_HELD |
| `OrderCommitted` | `order` | → ORDER_PLACED |
| `SessionClosed` | `closedAt` | → CLOSED (terminal happy) |
| `SessionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Session.initial("")` with `messages = List.of()`, all `Optional` fields as `Optional.empty()`, and `status = OPEN`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session` with `messages` truncated to the last 20 entries (to keep view row size bounded). Full message history is available via `GET /api/sessions/{id}` which reads directly from the entity.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint sorts and filters client-side.

## Task definition (`SessionTasks.java`)

```java
public final class SessionTasks {
  public static final Task<AssistantResponse> HANDLE_MESSAGE = Task
      .name("Handle customer message")
      .description("Read the session context and customer message; return an AssistantResponse with a reply and optional tool call")
      .resultConformsTo(AssistantResponse.class);

  private SessionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
