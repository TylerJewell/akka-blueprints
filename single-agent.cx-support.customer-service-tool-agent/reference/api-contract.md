# API contract — customer-service-tool-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `StartConversationRequest` | `201 { conversationId }` | `SupportEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{id}/messages` | `SendMessageRequest` | `202 { messageId }` | `SupportEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` (newest-first) | `SupportEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 ConversationRow` / `404` | `SupportEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `SupportEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartConversationRequest (request body)

```json
{
  "customerId": "CUST-1042",
  "messageText": "Where is my order ORD-5501?"
}
```

### SendMessageRequest (request body)

```json
{
  "messageText": "Can I get a refund for that order?"
}
```

### ConversationRow (response body)

```json
{
  "conversationId": "conv-7a3f...",
  "customerId": "CUST-1042",
  "turns": [
    {
      "message": {
        "messageId": "msg-001",
        "customerId": "CUST-1042",
        "text": "Where is my order ORD-5501?",
        "sentAt": "2026-06-28T14:00:00Z"
      },
      "reply": {
        "replyText": "Your order ORD-5501 shipped on 2026-06-25 via FedEx (tracking: FX-9921-4480). Estimated delivery is 2026-06-30.",
        "toolCalls": [
          {
            "toolName": "lookupOrder",
            "arguments": { "orderId": "ORD-5501" },
            "rawResult": "{ \"status\": \"SHIPPED\", \"carrier\": \"FedEx\", \"tracking\": \"FX-9921-4480\" }",
            "outcome": "ALLOWED",
            "blockReason": null
          }
        ],
        "outcome": "SENT",
        "repliedAt": "2026-06-28T14:00:22Z"
      }
    }
  ],
  "lastScreened": {
    "redactedReplyText": "Your order ORD-5501 shipped on 2026-06-25 via FedEx (tracking: FX-9921-4480). Estimated delivery is 2026-06-30.",
    "piiCategoriesFound": []
  },
  "status": "REPLIED",
  "openedAt": "2026-06-28T14:00:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### Blocked-tool-call example (within a turn's reply)

```json
{
  "toolCalls": [
    {
      "toolName": "processRefund",
      "arguments": { "orderId": "ORD-5501", "amount": "420.00" },
      "rawResult": "BLOCKED",
      "outcome": "BLOCKED",
      "blockReason": "Refund amount 420.00 exceeds auto-approve limit 250.0; escalate to human operator."
    },
    {
      "toolName": "openEscalation",
      "arguments": { "customerId": "CUST-1042", "reason": "refund above auto-approve threshold", "amount": "420.00" },
      "rawResult": "{ \"ticketId\": \"ESC-9910\", \"status\": \"OPEN\" }",
      "outcome": "ALLOWED",
      "blockReason": null
    }
  ]
}
```

### SSE event format

```
event: conversation-update
data: { "conversationId": "conv-7a3f...", "status": "REPLIED", "turns": [...], ... }
```

One event per state transition (`OPEN`, `ACTIVE`, `REPLIED`, `ESCALATED`, `RESOLVED`, `FAILED`). Clients reconcile by `conversationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `customerId` from the authenticated principal rather than the request body.
