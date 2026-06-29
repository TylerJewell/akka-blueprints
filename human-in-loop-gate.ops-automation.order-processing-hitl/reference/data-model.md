# Data model

Every record, event, enum, and view row the generated system defines.

## `Order` (OrderEntity state + OrdersView row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `id` | `String` | no | Order id; equals the workflow id and entity id |
| `customerId` | `Optional<String>` | yes | The customer placing the order |
| `status` | `OrderStatus` | no | Lifecycle status |
| `submittedAt` | `Optional<Instant>` | yes | When the order was first recorded |
| `lineItemsSummary` | `Optional<String>` | yes | Human-readable line-item summary from ValidationAgent |
| `estimatedTotal` | `Optional<String>` | yes | Formatted currency total from ValidationAgent |
| `riskFlags` | `Optional<String>` | yes | Comma-separated risk flags from ValidationAgent |
| `reviewedAt` | `Optional<Instant>` | yes | When the validation review was written |
| `approvedAt` | `Optional<Instant>` | yes | When the order was approved |
| `approvedBy` | `Optional<String>` | yes | Approver identity |
| `approverNote` | `Optional<String>` | yes | Approver note |
| `rejectedAt` | `Optional<Instant>` | yes | When the order was rejected |
| `rejectedBy` | `Optional<String>` | yes | Rejecter identity |
| `rejectReason` | `Optional<String>` | yes | Reject reason |
| `fulfilledAt` | `Optional<String>` | yes | ISO-8601 dispatch time |
| `trackingId` | `Optional<String>` | yes | Tracking id from FulfillmentAgent |

Every nullable lifecycle field is `Optional<T>` because `Order` is the view row type (Lesson 6). `emptyState()` returns `Order.initial("")` with no `commandContext()` reference (Lesson 3).

## `OrderStatus` enum

`PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `FULFILLED`.

## Events

| Event | Trigger | Effect |
|---|---|---|
| `OrderSubmitted` | `recordReview` after ValidationAgent returns | sets `submittedAt`, `reviewedAt`, `lineItemsSummary`, `estimatedTotal`, `riskFlags`, status → `PENDING_APPROVAL` |
| `OrderApproved` | `approve` command (human via API) | sets `approvedAt`, `approvedBy`, `approverNote`, status → `APPROVED` |
| `OrderRejected` | `reject` command (human via API) | sets `rejectedAt`, `rejectedBy`, `rejectReason`, status → `REJECTED` |
| `OrderFulfilled` | `recordFulfillment` after FulfillmentAgent returns | sets `fulfilledAt`, `trackingId`, status → `FULFILLED` |

## Domain records

- `OrderReview(String lineItems, String estimatedTotal, String riskFlags)` — ValidationAgent result.
- `ApprovalDecision(String approvedBy, String note)` — approve payload.
- `DispatchConfirmation(String trackingId, String dispatchedAt)` — FulfillmentAgent result.

## View row type

`OrdersView` rows are `Order`. One query: `getAllOrders` → `SELECT * AS orders FROM orders_view`. No `WHERE status` filter; callers filter by status client-side (Lesson 2).

## Task constants (OrderTasks.java)

- `VALIDATE` — `Task.name(...).resultConformsTo(OrderReview.class)`.
- `FULFILL` — `Task.name(...).resultConformsTo(DispatchConfirmation.class)`.
