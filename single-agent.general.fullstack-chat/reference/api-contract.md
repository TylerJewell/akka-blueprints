# API contract — gemini-fullstack

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `CreateConversationRequest` | `201 { conversationId }` | `ChatEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ Conversation... ]` (newest-first) | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ChatEndpoint` ← `ConversationView` |
| `POST` | `/api/conversations/{id}/messages` | `SendMessageRequest` | `202 { messageId, workflowId }` | `ChatEndpoint` → `ConversationEntity`, `ConversationWorkflow` |
| `GET` | `/api/conversations/{id}/sse` | — | `text/event-stream` | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### CreateConversationRequest (request body)

```json
{
  "title": "Akka entity vs workflow",
  "submittedBy": "user-42"
}
```

### SendMessageRequest (request body)

```json
{
  "userMessage": "Can a workflow call an entity?",
  "submittedBy": "user-42"
}
```

### Conversation (response body)

```json
{
  "conversationId": "c-7a3f...",
  "title": "Akka entity vs workflow",
  "messages": [
    {
      "messageId": "m-001",
      "role": "USER",
      "content": "What is the difference between a workflow and an entity in Akka?",
      "tokenCount": null,
      "createdAt": "2026-06-28T14:00:00Z"
    },
    {
      "messageId": "m-002",
      "role": "AGENT",
      "content": "An entity stores durable state as an event log; a workflow orchestrates multi-step processes with explicit timeouts and recovery.",
      "tokenCount": 28,
      "createdAt": "2026-06-28T14:00:18Z"
    },
    {
      "messageId": "m-003",
      "role": "USER",
      "content": "Can a workflow call an entity?",
      "tokenCount": null,
      "createdAt": "2026-06-28T14:01:00Z"
    }
  ],
  "status": "AWAITING_REPLY",
  "createdAt": "2026-06-28T14:00:00Z",
  "lastActivityAt": "2026-06-28T14:01:00Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `tokenCount` is `null` for `USER` messages and a non-null integer for `AGENT` messages.

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-7a3f...", "status": "REPLY_RECORDED", "messages": [ ... ], ... }
```

One event per state transition (`CREATED`, `ACTIVE`, `AWAITING_REPLY`, `REPLY_RECORDED`, `FAILED`). The `data` field always carries the full `Conversation` row at the moment of transition, so a late-joining client never needs to replay events. Clients reconcile by `conversationId`.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
