# API contract — react-chatbot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `StartConversationRequest` | `201 { conversationId }` | `ChatEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{id}/messages` | `SendMessageRequest` | `201 { turnId }` / `409` | `ChatEndpoint` → `ConversationEntity` + `ConversationWorkflow` |
| `GET` | `/api/conversations` | — | `200 [ Conversation... ]` (newest-first) | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartConversationRequest (request body)

```json
{
  "title": "Support inquiry — order tracking",
  "createdBy": "user-4821"
}
```

### SendMessageRequest (request body)

```json
{
  "content": "Hi, where is my order ORD-003?",
  "sentBy": "user-4821"
}
```

### Conversation (response body)

```json
{
  "conversationId": "c-7a3...",
  "title": "Support inquiry — order tracking",
  "createdBy": "user-4821",
  "turns": [
    {
      "turnId": "t-1f2...",
      "message": {
        "turnId": "t-1f2...",
        "content": "Hi, where is my order ORD-003?",
        "sentBy": "user-4821",
        "sentAt": "2026-06-28T14:05:00Z"
      },
      "reply": {
        "content": "Your order ORD-003 is currently in transit and is estimated to arrive by July 2, 2026.",
        "toolCallTrace": [
          {
            "toolName": "get_order_status",
            "inputSummary": "orderId = ORD-003",
            "resultSummary": "status = IN_TRANSIT, estimatedDelivery = 2026-07-02"
          }
        ],
        "repliedAt": "2026-06-28T14:05:18Z"
      },
      "status": "COMPLETED",
      "failureReason": null,
      "startedAt": "2026-06-28T14:05:00Z",
      "finishedAt": "2026-06-28T14:05:18Z"
    }
  ],
  "status": "ACTIVE",
  "createdAt": "2026-06-28T14:05:00Z",
  "lastActiveAt": "2026-06-28T14:05:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### 409 Conflict response

```json
{
  "error": "conflict",
  "detail": "Conversation c-7a3... already has a PROCESSING turn t-1f2...; wait for it to complete before sending another message."
}
```

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-7a3...", "status": "ACTIVE", "turns": [...], ... }
```

One event per state transition across all conversations. The event always carries the full `Conversation` row at the moment of transition (including all turns up to that point), so a late-joining client never needs to replay prior events to reconstruct state.

Clients reconcile by `conversationId`. Within a conversation, the most recent `turns` entry reflects the current turn's status (`PROCESSING`, `COMPLETED`, or `FAILED`).

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `createdBy` / `sentBy` from the authenticated principal rather than the request body.
