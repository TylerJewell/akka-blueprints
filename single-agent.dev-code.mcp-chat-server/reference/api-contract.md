# API contract — mcp-chat-server

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `CreateConversationRequest` | `201 { conversationId }` | `ChatEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` (newest-first) | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 ConversationRow` / `404` | `ChatEndpoint` ← `ConversationView` |
| `POST` | `/api/conversations/{id}/messages` | `SendMessageRequest` | `201 { turnId }` | `ChatEndpoint` → `ConversationEntity`, `ChatWorkflow` |
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
  "title": "Debug session — NullPointerException in Router",
  "createdBy": "dev-user-42"
}
```

### SendMessageRequest (request body)

```json
{
  "text": "Search for the Akka HTTP routing DSL documentation.",
  "sentBy": "dev-user-42"
}
```

### ConversationRow (response body — list and get)

```json
{
  "conversationId": "conv-7fa...",
  "title": "Debug session — NullPointerException in Router",
  "createdBy": "dev-user-42",
  "status": "ACTIVE",
  "createdAt": "2026-06-28T09:00:00Z",
  "lastActiveAt": "2026-06-28T09:12:05Z",
  "turns": [
    {
      "turnId": "turn-3bc...",
      "message": {
        "turnId": "turn-3bc...",
        "conversationId": "conv-7fa...",
        "text": "Search for the Akka HTTP routing DSL documentation.",
        "sentBy": "dev-user-42",
        "sentAt": "2026-06-28T09:12:00Z"
      },
      "reply": {
        "replyText": "Here are the top results for 'Akka HTTP routing DSL':\n- doc.akka.io/akka-http/routing-dsl.html\n- doc.akka.io/akka-http/path-matchers.html",
        "toolCallsAudited": [
          {
            "toolName": "search",
            "serverUrl": "http://localhost:9900/mcp",
            "inputSummary": "query: 'Akka HTTP routing DSL documentation'",
            "outcome": "ALLOWED",
            "calledAt": "2026-06-28T09:12:01Z"
          }
        ],
        "iterationsUsed": 1,
        "repliedAt": "2026-06-28T09:12:04Z"
      },
      "status": "REPLIED",
      "failureReason": null,
      "startedAt": "2026-06-28T09:12:00Z",
      "finishedAt": "2026-06-28T09:12:05Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: conversation-update
data: { "conversationId": "conv-7fa...", "turnId": "turn-3bc...", "status": "REPLIED", "reply": { ... }, ... }
```

One event per turn state transition (`RECEIVED`, `RUNNING`, `REPLIED`, `FAILED`). Clients subscribe to `/api/conversations/{id}/sse` for per-conversation updates. An event always carries the full `ConversationRow` at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `sentBy` from the authenticated principal rather than the request body. The `conversationId` in the path serves as the resource scope; all turns under a conversation share the same access boundary.
