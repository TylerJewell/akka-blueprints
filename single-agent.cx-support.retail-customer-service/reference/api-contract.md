# API contract — retail-customer-service

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `CustomerServiceEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{id}/turn` | `SendTurnRequest` | `202 { turnId }` | `CustomerServiceEndpoint` → `SessionWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `CustomerServiceEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `CustomerServiceEndpoint` ← `SessionEntity` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `CustomerServiceEndpoint` ← `SessionView` |
| `GET` | `/api/orders/{id}` | — | `200 Order` / `404` | `CustomerServiceEndpoint` ← `OrderView` |
| `GET` | `/api/products` | — | `200 [ Product... ]` | `CustomerServiceEndpoint` (catalog) |
| `GET` | `/api/products/{id}` | — | `200 Product` / `404` | `CustomerServiceEndpoint` (catalog) |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "customerId": "customer-881",
  "customerMessage": "Do you have any pet-safe succulents?",
  "orderId": null
}
```

### SendTurnRequest (request body)

```json
{
  "customerMessage": "Can you cancel order ORD-1042?",
  "orderId": "ORD-1042"
}
```

### Session (response body — GET /api/sessions/{id})

```json
{
  "sessionId": "s-3af...",
  "customerId": "customer-881",
  "turns": [
    {
      "turnId": "t-001",
      "customerMessage": "Do you have any pet-safe succulents?",
      "reply": {
        "message": "Haworthia fasciata is non-toxic to cats and dogs and thrives in indirect light. We have it in stock at $8.99.",
        "orderChange": null,
        "model": "claude-sonnet-4-6",
        "repliedAt": "2026-06-28T14:00:05Z"
      },
      "orderContext": null,
      "status": "REPLIED",
      "startedAt": "2026-06-28T14:00:00Z",
      "completedAt": "2026-06-28T14:00:05Z"
    },
    {
      "turnId": "t-002",
      "customerMessage": "Can you cancel order ORD-1042?",
      "reply": {
        "message": "Done — order ORD-1042 has been cancelled. You'll receive a confirmation email shortly.",
        "orderChange": {
          "orderId": "ORD-1042",
          "changeType": "cancel",
          "field": "status",
          "newValue": "CANCELLED"
        },
        "model": "claude-sonnet-4-6",
        "repliedAt": "2026-06-28T14:01:10Z"
      },
      "orderContext": "ORD-1042",
      "status": "REPLIED",
      "startedAt": "2026-06-28T14:01:00Z",
      "completedAt": "2026-06-28T14:01:11Z"
    }
  ],
  "status": "ACTIVE",
  "openedAt": "2026-06-28T14:00:00Z",
  "closedAt": null
}
```

### Order (response body — GET /api/orders/{id})

```json
{
  "orderId": "ORD-1042",
  "customerId": "customer-881",
  "lineItems": [
    {
      "productId": "prod-haworthia-001",
      "productName": "Haworthia fasciata (Zebra Plant)",
      "quantity": 2,
      "unitPriceUsd": 8.99
    }
  ],
  "shippingAddress": "123 Garden Lane, Springfield, IL 62701",
  "status": "PENDING",
  "placedAt": "2026-06-27T09:00:00Z",
  "shippedAt": null,
  "deliveredAt": null,
  "modifications": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SessionRow (response body — GET /api/sessions list item)

```json
{
  "sessionId": "s-3af...",
  "customerId": "customer-881",
  "lastTurnStatus": "REPLIED",
  "lastReplyPreview": "Haworthia fasciata is non-toxic to cats...",
  "turnCount": 2,
  "status": "ACTIVE",
  "openedAt": "2026-06-28T14:00:00Z"
}
```

### SSE event format

```
event: session-update
data: { "sessionId": "s-3af...", "status": "ACTIVE", "lastTurnStatus": "REPLIED", ... }
```

One event per turn transition (`PENDING`, `REPLIED`, `GUARDRAIL_BLOCKED`, `FAILED`) and one on session status transitions (`OPEN`, `ACTIVE`, `CLOSED`). Clients reconcile by `sessionId` and `turnId`; an event always carries the full `SessionRow` at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `CustomerServiceEndpoint` with their auth middleware and populate `customerId` from the authenticated principal rather than the request body.
