# API contract — rag-support-bot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `SubmitQueryRequest` | `201 { conversationId }` | `ConversationEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ Conversation... ]` (newest-first) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "rawQuery": "Hi, I ordered something last week (order #ORD-98765, jane.doe@example.com) but haven't received a shipping notification. What's the status?",
  "topicFilter": "ORDERS",
  "submittedBy": "session-a3f92c"
}
```

`topicFilter` accepts `ANY`, `ORDERS`, `RETURNS`, `ACCOUNT`, `TECHNICAL`. Defaults to `ANY` if omitted.

### Conversation (response body)

```json
{
  "conversationId": "c-7e4a...",
  "query": {
    "conversationId": "c-7e4a...",
    "rawQuery": "Hi, I ordered something last week (order #ORD-98765, jane.doe@example.com) but haven't received a shipping notification. What's the status?",
    "topicFilter": "ORDERS",
    "submittedBy": "session-a3f92c",
    "receivedAt": "2026-06-28T14:00:00Z"
  },
  "sanitized": {
    "redactedQuery": "Hi, I ordered something last week (order [REDACTED-ORDER-ID], [REDACTED-EMAIL]) but haven't received a shipping notification. What's the status?",
    "piiCategoriesFound": ["order-id", "email"]
  },
  "retrieved": {
    "passages": [
      {
        "passageId": "orders-faq-p1",
        "articleId": "orders-faq",
        "articleTitle": "Order Status FAQ",
        "passageText": "After your order ships, you will receive a shipping confirmation email within 24 hours. If you have not received one after 48 hours, contact support.",
        "topic": "ORDERS"
      },
      {
        "passageId": "orders-faq-p2",
        "articleId": "orders-faq",
        "articleTitle": "Order Status FAQ",
        "passageText": "Order status can be checked in your account under My Orders. Tracking numbers are activated 1–2 business days after dispatch.",
        "topic": "ORDERS"
      }
    ],
    "passageCount": 2
  },
  "reply": {
    "answerText": "Shipping confirmations are sent within 24 hours of dispatch. If it has been over 48 hours, a specialist can look up your specific order — please use the escalation option.",
    "sourcePassageIds": ["orders-faq-p1"],
    "escalation": false,
    "escalationReason": "",
    "outcome": "PARTIALLY_ANSWERED",
    "repliedAt": "2026-06-28T14:00:18Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Answer cites a valid passage and stays within length; one additional passage could strengthen the tracking detail.",
    "evaluatedAt": "2026-06-28T14:00:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-7e4a...", "status": "REPLY_RECORDED", "reply": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `RETRIEVING`, `REPLYING`, `REPLY_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `conversationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated session rather than the request body.
