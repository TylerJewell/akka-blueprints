# Data model — games-sales-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GameTitle` | `titleId` | `String` | no | Stable catalog id (e.g. `steel-hollow-pc`). |
| | `name` | `String` | no | Display name. |
| | `platform` | `String` | no | e.g. `PC`, `PlayStation`, `Xbox`, `Switch`. |
| | `genre` | `String` | no | e.g. `action-rpg`, `sports`, `strategy`. |
| | `priceUsd` | `double` | no | Current retail price. |
| | `shortPitch` | `String` | no | One-sentence catalog description. Used by the guardrail to validate promotional claims. |
| | `inStock` | `boolean` | no | Whether the title is currently available. |
| `OrderLine` | `orderId` | `String` | no | Stable order id. |
| | `titleId` | `String` | no | References a `GameTitle.titleId`. |
| | `titleName` | `String` | no | Denormalised for display without catalog lookup. |
| | `paidUsd` | `double` | no | Amount paid at purchase time. |
| | `purchasedAt` | `Instant` | no | Purchase timestamp. |
| `ShopperContext` | `sessionId` | `String` | no | Current session. |
| | `shopperId` | `String` | no | Shopper identifier. |
| | `recentOrders` | `List<OrderLine>` | no | Last 5 orders; may be empty list. |
| | `previousTurnIds` | `List<String>` | no | Prior turn ids in this session for threading. |
| `TurnRequest` | `turnId` | `String` | no | UUID minted by `SessionEndpoint`. |
| | `question` | `String` | no | Shopper's natural-language question. |
| | `context` | `ShopperContext` | no | Session context at time of question. |
| | `askedAt` | `Instant` | no | When the endpoint received the request. |
| `Recommendation` | `titleId` | `String` | no | MUST exist in `CatalogIndex`. |
| | `name` | `String` | no | Exact title name from catalog. |
| | `platform` | `String` | no | Exact platform string from catalog. |
| | `priceUsd` | `double` | no | Exact price from catalog. |
| | `pitch` | `String` | no | Agent-authored one-sentence pitch. |
| `AssistantResponse` | `answer` | `String` | no | Plain-language 1–4 sentence answer. |
| | `recommendations` | `List<Recommendation>` | no | 0–5 entries; empty list if not a browsing question. |
| | `orders` | `List<OrderLine>` | no | Empty list if not an order-history question. |
| | `status` | `ResponseStatus` | no | Always `ANSWERED` on a valid agent response. |
| | `answeredAt` | `Instant` | no | When the agent returned. |
| `Turn` | `turnId` | `String` | no | — |
| | `sessionId` | `String` | no | — |
| | `request` | `TurnRequest` | no | Always present once turn is created. |
| | `response` | `Optional<AssistantResponse>` | yes | Populated after `TurnAnswered`. |
| | `status` | `TurnStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `TurnStarted` emitted. |
| | `answeredAt` | `Optional<Instant>` | yes | Terminal-state timestamp for this turn. |
| `Session` (entity state) | `sessionId` | `String` | no | — |
| | `shopperId` | `String` | no | — |
| | `turns` | `List<Turn>` | no | Ordered list; may be empty on `CREATED`. |
| | `status` | `SessionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `SessionCreated` emitted. |
| | `closedAt` | `Optional<Instant>` | yes | Populated after `SessionClosed` or `SessionFailed`. |

Every nullable field is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ResponseStatus`: `ANSWERED`, `GUARDRAIL_REJECTED`, `AGENT_FAILED`.
`TurnStatus`: `PENDING`, `ANSWERED`, `GUARDRAIL_REJECTED`, `FAILED`.
`SessionStatus`: `CREATED`, `ACTIVE`, `CLOSED`, `FAILED`.

## Events (`SessionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `SessionCreated` | `shopperId` | → CREATED |
| `TurnStarted` | `request: TurnRequest` | CREATED → ACTIVE (first turn); ACTIVE stays ACTIVE |
| `TurnAnswered` | `turnId, response: AssistantResponse` | ACTIVE stays ACTIVE |
| `TurnRejected` | `turnId, reason: String` | ACTIVE stays ACTIVE |
| `TurnFailed` | `turnId, reason: String` | ACTIVE stays ACTIVE |
| `SessionClosed` | — | ACTIVE → CLOSED (terminal happy) |
| `SessionFailed` | `reason: String` | CREATED/ACTIVE → FAILED (terminal) |

`emptyState()` returns `Session.initial("")` with empty turns list, status `CREATED`, and `closedAt` as `Optional.empty()`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`SessionRow` mirrors `Session` with turn count, last-turn status, and last-turn answeredAt — without full turn bodies to keep the view lean. Full turn content is available via `GET /api/sessions/{id}`.

The view declares ONE query: `getAllSessions: SELECT * AS sessions FROM session_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`SalesTasks.java`)

```java
public final class SalesTasks {
  public static final Task<AssistantResponse> ASSIST_SHOPPER = Task
      .name("Assist shopper")
      .description("Answer the shopper's question about the video games catalog and return an " +
                   "AssistantResponse with recommendations and optional order history")
      .resultConformsTo(AssistantResponse.class);

  private SalesTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## CatalogIndex

`CatalogIndex` is an in-process singleton (not an Akka primitive). It wraps a `ConcurrentHashMap<String, GameTitle>` and exposes:

- `void upsert(GameTitle title)` — called by `CatalogConsumer` on `CatalogUpdated` events.
- `boolean contains(String titleId)` — called by `ResponseGuardrail` for title-existence checks.
- `Optional<GameTitle> get(String titleId)` — called by `ResponseGuardrail` to retrieve `shortPitch` for promotional-claim validation.

Pre-populated at startup from `src/main/resources/sample-events/catalog.jsonl` inside `CatalogConsumer`'s startup hook, before any shopper session begins.
