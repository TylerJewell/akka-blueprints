# API contract — whatsapp-order-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `OrderEndpoint` → `SessionEntity` |
| `POST` | `/api/sessions/{id}/turns` | `SubmitTurnRequest` | `201 { turnId }` | `OrderEndpoint` → `SessionEntity`, `OrderWorkflow` |
| `GET` | `/api/sessions` | — | `200 [ SessionRow... ]` (newest-first) | `OrderEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 SessionRow` / `404` | `OrderEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `OrderEndpoint` ← `SessionView` |
| `POST` | `/api/hitl/{orderId}/approve` | — | `200 { ok: true }` | `OrderEndpoint` → `SessionEntity` |
| `POST` | `/api/hitl/{orderId}/reject` | `{ reason: String }` | `200 { ok: true }` | `OrderEndpoint` → `SessionEntity` |
| `GET` | `/api/orders` | — | `200 [ Order... ]` | `OrderEndpoint` ← `SessionView` |
| `GET` | `/api/orders/{id}` | — | `200 Order` / `404` | `OrderEndpoint` ← `SessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `OrderEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `OrderEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `OrderEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "customerId": "cust-789",
  "whatsappPhoneNumber": "+15558675309"
}
```

### SubmitTurnRequest (request body)

```json
{
  "customerMessage": "Hi, I'd like to order 2 Blue Widgets please"
}
```

### SessionRow (response body)

```json
{
  "sessionId": "sess-4a1...",
  "customerId": "cust-789",
  "status": "COMPLETING",
  "turns": [
    {
      "turnId": "turn-001",
      "customerMessage": "Hi, I'd like to order 2 Blue Widgets please",
      "agentReply": "Done! Your order #ORD-00482 for 2 × Blue Widget has been placed.",
      "piiCategoriesRedacted": ["phone"]
    }
  ],
  "pendingOrder": null,
  "lastTurnEval": {
    "score": 4,
    "rationale": "Reply references tool result; appropriate length; no redacted markers exposed.",
    "evaluatedAt": "2026-06-28T10:00:16Z"
  },
  "createdAt": "2026-06-28T10:00:00Z",
  "closedAt": null
}
```

### Order (response body)

```json
{
  "orderId": "ORD-00482",
  "sessionId": "sess-4a1...",
  "status": "CONFIRMED",
  "request": {
    "orderId": "ORD-00482",
    "sessionId": "sess-4a1...",
    "customerId": "cust-789",
    "items": [
      {
        "sku": "SKU-1001",
        "productName": "Blue Widget",
        "quantity": 2,
        "unitPrice": 24.99,
        "lineTotal": 49.98
      }
    ],
    "orderTotal": 49.98,
    "currency": "USD",
    "deliveryAddress": "[REDACTED-ADDRESS]",
    "requestedAt": "2026-06-28T10:00:15Z"
  },
  "createdAt": "2026-06-28T10:00:15Z",
  "confirmedAt": "2026-06-28T10:00:16Z",
  "cancelledAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "sess-4a1...", "status": "AWAITING_APPROVAL", "pendingOrder": { ... }, ... }
```

One event per state transition (`IDLE`, `ACTIVE`, `AWAITING_APPROVAL`, `COMPLETING`, `CLOSED`, `FAILED`). Clients reconcile by `sessionId`; an event always carries the full `SessionRow` at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware. The `customerId` field should be set from the authenticated principal rather than the request body. The HITL endpoints (`/api/hitl/*`) require operator-level privileges in a real deployment.
