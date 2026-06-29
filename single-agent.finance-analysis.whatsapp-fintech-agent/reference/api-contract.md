# API contract — whatsapp-fintech-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/messages` | `ReceiveMessageRequest` | `201 { messageId }` | `MessageEndpoint` → `MessageEntity` |
| `GET` | `/api/messages` | — | `200 [ Message... ]` (newest-first) | `MessageEndpoint` ← `MessageView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `MessageEndpoint` ← `MessageView` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `MessageEndpoint` ← `MessageView` |
| `POST` | `/api/messages/{id}/approve` | `HitlActionRequest` | `204` | `MessageEndpoint` → `MessageEntity` |
| `POST` | `/api/messages/{id}/reject` | `HitlActionRequest` | `204` | `MessageEndpoint` → `MessageEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### ReceiveMessageRequest (request body)

```json
{
  "customerId": "cust-0001",
  "accountId": "acc-0042",
  "rawText": "Hi, can you transfer $1200 to acc-0099 for my rent please? My number is 07700 900123.",
  "channelMessageId": "whatsapp-msg-7fa3b2"
}
```

### HitlActionRequest (request body for approve/reject)

```json
{
  "operatorId": "op-ops-007",
  "note": "Verified customer via callback; approved."
}
```

### Message (response body)

```json
{
  "messageId": "m-4d9a...",
  "inbound": {
    "messageId": "m-4d9a...",
    "customerId": "cust-0001",
    "accountId": "acc-0042",
    "rawText": "Hi, can you transfer $1200 to acc-0099 for my rent please? My number is 07700 900123.",
    "channelMessageId": "whatsapp-msg-7fa3b2",
    "receivedAt": "2026-06-28T09:00:00Z"
  },
  "sanitized": {
    "redactedText": "Hi, can you transfer $1200 to [REDACTED-ACCOUNT-NUMBER] for my rent please? My number is [REDACTED-PHONE].",
    "piiCategoriesFound": ["phone", "account-number"]
  },
  "response": {
    "intent": "FUND_TRANSFER",
    "responseText": "Your transfer of $1,200.00 to the requested account has been submitted for approval.",
    "toolCallsMade": [
      {
        "toolName": "transfer_funds",
        "args": {
          "fromAccountId": "acc-0042",
          "toAccountId": "acc-0099",
          "amount": "1200.00",
          "currency": "USD",
          "description": "Rent payment"
        },
        "result": "txn-ref-5541"
      }
    ],
    "pendingTransaction": {
      "fromAccountId": "acc-0042",
      "toAccountId": "acc-0099",
      "amount": 1200.00,
      "currency": "USD",
      "description": "Rent payment"
    },
    "decidedAt": "2026-06-28T09:00:22Z"
  },
  "hitlDecision": {
    "operatorId": "op-ops-007",
    "outcome": "APPROVED",
    "note": "Verified customer via callback; approved.",
    "decidedAt": "2026-06-28T09:02:15Z"
  },
  "status": "EXECUTED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:02:16Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: message-update
data: { "messageId": "m-4d9a...", "status": "AWAITING_APPROVAL", "response": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `RESPONDING`, `RESPONSE_READY`, `AWAITING_APPROVAL`, `EXECUTED`, `HITL_REJECTED`, `FAILED`). Clients reconcile by `messageId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, derive `customerId` from the authenticated session rather than the request body, and add a role check on the `/approve` and `/reject` endpoints (operator role only).
