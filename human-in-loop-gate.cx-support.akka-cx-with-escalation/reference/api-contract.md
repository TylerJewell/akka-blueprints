# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/conversations` | `{ "customerMessage": "string" }` | `{ "conversationId": "uuid" }` | SupportEndpoint → SupportWorkflow |
| POST | `/api/conversations/{conversationId}/escalate` | `{ "reason": "string" }` | `200` \| `404` | SupportEndpoint → ConversationEntity |
| POST | `/api/conversations/{conversationId}/accept-escalation` | `{ "acceptedBy": "string" }` | `200` \| `404` | SupportEndpoint → ConversationEntity |
| POST | `/api/conversations/{conversationId}/resolve` | `{ "note": "string" }` | `200` \| `404` | SupportEndpoint → ConversationEntity |
| GET | `/api/conversations` | — | `{ "conversations": [Conversation, ...] }` | SupportEndpoint → ConversationsView |
| GET | `/api/conversations/{conversationId}` | — | `Conversation` \| `404` | SupportEndpoint → ConversationEntity |
| GET | `/api/conversations/sse` | — | SSE stream of `Conversation` | SupportEndpoint → ConversationsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | SupportEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | SupportEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | SupportEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Conversation` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "customerMessage": "string or null",
  "status": "ACTIVE | ESCALATING | ESCALATED | RESOLVED",
  "openedAt": "ISO-8601 or null",
  "agentReply": "string or null",
  "repliedAt": "ISO-8601 or null",
  "escalationReason": "string or null",
  "escalationRequestedAt": "ISO-8601 or null",
  "acceptedBy": "string or null",
  "acceptedAt": "ISO-8601 or null",
  "handoffSummary": "string or null",
  "handoffPriority": "low | medium | high | urgent | null",
  "resolvedAt": "ISO-8601 or null",
  "resolveNote": "string or null"
}
```

## SSE event format

`GET /api/conversations/sse` streams the `ConversationsView` rows. Each message:

```
event: conversation
data: { ...Conversation... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
