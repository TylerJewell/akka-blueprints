# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Served by `ChatEndpoint` (`/api/*`) and `AppEndpoint` (`/`, `/app/*`).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/conversations` | `{ "topic": string }` | `{ "conversationId": string }` | ChatEndpoint → InboundTopicQueue |
| GET | `/api/conversations` | — | `{ "conversations": [Conversation, ...] }` | ChatEndpoint → TurnView |
| GET | `/api/conversations/{id}` | — | `Conversation` \| 404 | ChatEndpoint → TurnView |
| GET | `/api/conversations/sse` | — | SSE stream of `Conversation` | ChatEndpoint → TurnView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ChatEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ChatEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ChatEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static resource | AppEndpoint |

The optional `?status=` filter on `GET /api/conversations` is applied client-side in the endpoint after `TurnView.getAllConversations` (Akka cannot auto-index the enum column — Lesson 2).

## Payload shapes

`Conversation` (the `TurnView` row; lifecycle fields are nullable — `Optional` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "topic": "string or null",
  "status": "IN_PROGRESS | APPROVED | MAX_ROUNDS_REACHED | ESCALATED | CLOSED",
  "currentSpeaker": "WRITER | EDITOR | null",
  "round": 1,
  "turns": [
    {
      "speaker": "WRITER | EDITOR",
      "content": "string",
      "round": 1,
      "evalScore": 0.0,
      "blocked": false,
      "blockedReason": "string or null",
      "at": "ISO-8601"
    }
  ],
  "latestDraft": "string or null",
  "latestCritique": "string or null",
  "createdAt": "ISO-8601 or null",
  "approvedAt": "ISO-8601 or null",
  "escalatedAt": "ISO-8601 or null",
  "closedAt": "ISO-8601 or null"
}
```

`evalScore` is `null` until the eval consumer scores an Editor turn.

## SSE event format

`GET /api/conversations/sse` streams `text/event-stream`. Each event's `data:` line is one `Conversation` JSON object (the full current state), emitted whenever a `ConversationEntity` event updates the `TurnView` row. The browser replaces its cached copy by `id`.

```
event: conversation
data: { "id": "...", "status": "IN_PROGRESS", "turns": [ ... ], ... }

```
