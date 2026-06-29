# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/orders` | `{ "customerId": "string", "lineItems": "string" }` | `{ "orderId": "uuid" }` | OrderEndpoint → OrderWorkflow |
| POST | `/api/orders/{orderId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | OrderEndpoint → OrderEntity |
| POST | `/api/orders/{orderId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | OrderEndpoint → OrderEntity |
| GET | `/api/orders` | — | `{ "orders": [Order, ...] }` | OrderEndpoint → OrdersView |
| GET | `/api/orders/{orderId}` | — | `Order` \| `404` | OrderEndpoint → OrderEntity |
| GET | `/api/orders/sse` | — | SSE stream of `Order` | OrderEndpoint → OrdersView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | OrderEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | OrderEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | OrderEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Order` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "customerId": "string or null",
  "status": "PENDING_APPROVAL | APPROVED | REJECTED | FULFILLED",
  "submittedAt": "ISO-8601 or null",
  "lineItemsSummary": "string or null",
  "estimatedTotal": "string or null",
  "riskFlags": "string or null",
  "reviewedAt": "ISO-8601 or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "fulfilledAt": "ISO-8601 or null",
  "trackingId": "string or null"
}
```

## SSE event format

`GET /api/orders/sse` streams the `OrdersView` rows. Each message:

```
event: order
data: { ...Order... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
