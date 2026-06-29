# API contract — entity-workflow-chat

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `StartConversationRequest` | `201 { conversationId }` | `ChatEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{id}/messages` | `SendMessageRequest` | `202 { messageId }` | `ChatEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{id}/close` | `{}` | `204` | `ChatEndpoint` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` (newest-first) | `ChatEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 ConversationRow` / `404` | `ChatEndpoint` ← `ConversationView` |
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
  "topic": "Duplicate charge on my June statement",
  "userId": "customer-7821"
}
```

### SendMessageRequest (request body)

```json
{
  "text": "I was charged twice on June 14th for order #A-4401. Can you look into that?"
}
```

### ConversationRow (response body)

```json
{
  "conversationId": "conv-3a9...",
  "topic": "Duplicate charge on my June statement",
  "userId": "customer-7821",
  "turns": [
    {
      "turnId": "turn-1",
      "userMessage": {
        "messageId": "turn-1",
        "redactedText": "I was charged twice on June 14th for order #A-4401. Can you look into that?",
        "piiCategoriesFound": [],
        "sanitizedAt": "2026-06-28T14:10:01Z"
      },
      "agentReply": {
        "messageId": "turn-1",
        "replyText": "I can see two charges dated 14 June for order #A-4401. The duplicate was reversed on 16 June; please allow 3–5 business days for it to appear on your statement.",
        "suggestClose": false,
        "repliedAt": "2026-06-28T14:10:08Z"
      },
      "status": "COMPLETED"
    }
  ],
  "summaries": [],
  "status": "OPEN",
  "createdAt": "2026-06-28T14:10:00Z",
  "closedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

A conversation with compacted history would include a non-empty `summaries` array:

```json
{
  "summaries": [
    {
      "summaryText": "- Customer reported duplicate charge on 14 June for order #A-4401.\n- Agent confirmed reversal on 16 June; customer acknowledged.",
      "turnsCompacted": 4,
      "compactedAt": "2026-06-28T14:25:00Z"
    }
  ]
}
```

### SSE event format

```
event: conversation-update
data: { "conversationId": "conv-3a9...", "status": "OPEN", "turns": [...], ... }
```

One event per state transition (`OPEN`, `AGENT_THINKING`, `COMPACTING`, `CLOSED`, `FAILED`). Clients reconcile by `conversationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and source `userId` from the authenticated principal rather than the request body.
