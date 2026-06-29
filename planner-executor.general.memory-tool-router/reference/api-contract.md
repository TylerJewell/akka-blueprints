# API contract — memory-tool-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `{ "sessionId": String, "text": String, "sentBy"?: String }` | `202 { "conversationId": String }` | `ConversationEndpoint` → `MessageQueue` |
| `GET` | `/api/conversations` | — | `200 [ ConversationRow... ]` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ConversationEndpoint` ← `ConversationEntity` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` (one event per conversation change) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}/memory` | — | `200 MemorySnapshot` / `404` | `ConversationEndpoint` ← `MemoryEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `ConversationEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `ConversationEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `ConversationEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Conversation (full form, returned by `GET /api/conversations/{id}`)

```json
{
  "conversationId": "c-9a1…",
  "sessionId": "s-user42",
  "status": "PROCESSING",
  "messageHistory": [
    "What is 2+2?",
    "The result is 4.",
    "What do you know about Akka workflows?"
  ],
  "turnLog": {
    "entries": [
      {
        "turnNumber": 1,
        "messageText": "What is 2+2?",
        "toolUsed": "CALCULATOR",
        "verdict": "OK",
        "reply": "2 + 2 = 4",
        "blocker": null,
        "completedAt": "2026-06-28T09:14:02Z"
      },
      {
        "turnNumber": 2,
        "messageText": "What do you know about Akka workflows?",
        "toolUsed": "KNOWLEDGE_BASE",
        "verdict": "OK",
        "reply": "Akka workflows are durable, resumable state machines…",
        "blocker": null,
        "completedAt": "2026-06-28T09:14:11Z"
      }
    ]
  },
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:13:58Z",
  "finishedAt": null
}
```

### ConversationRow (list form, returned by `GET /api/conversations`)

Same fields as `Conversation`, but `messageHistory` is truncated to the last 5 messages, and `turnLog.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count. The UI fetches the full conversation by id on row expand.

### MemorySnapshot (returned by `GET /api/conversations/{id}/memory`)

```json
{
  "sessionId": "s-user42",
  "items": [
    {
      "key": "user-preference-verbosity",
      "value": "prefers concise answers",
      "kind": "PREFERENCE",
      "recordedAt": "2026-06-28T09:14:02Z"
    },
    {
      "key": "user-email",
      "value": "[REDACTED:email]",
      "kind": "ENTITY_MENTION",
      "recordedAt": "2026-06-28T09:14:11Z"
    }
  ]
}
```

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-9a1…", "status": "PROCESSING", "lastTurnNumber": 2, ... }
```

One event per state transition or turn completion. Clients reconcile by `conversationId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating unexpected tool routing", "haltedAt": "2026-06-28T09:15:30Z" }
```
