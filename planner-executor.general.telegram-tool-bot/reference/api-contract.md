# API contract — telegram-tool-bot

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `{ "text": String, "chatId"?: String }` | `202 { "sessionId": String }` | `SessionEndpoint` → `MessageQueue` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionEntity` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` (one event per session change) | `SessionEndpoint` ← `SessionView` |
| `POST` | `/api/control/pause` | `{ "reason": String }` | `200 { "paused": true, "reason": String }` | `SessionEndpoint` → `BotControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "paused": false }` | `SessionEndpoint` → `BotControlEntity` |
| `GET` | `/api/control` | — | `200 { "paused": boolean, "reason"?: String, "pausedAt"?: Instant }` | `SessionEndpoint` ← `BotControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Session (full form, returned by `GET /api/sessions/{id}`)

```json
{
  "sessionId": "s-9e4…",
  "chatId": "chat-112233",
  "text": "Look up the latest Akka announcements and give me a quick summary.",
  "status": "EXECUTING",
  "ledger": {
    "intent": "the user wants a summary of recent Akka announcements",
    "facts": ["the user is asking about Akka", "they want a concise summary"],
    "toolPlan": [
      "Web-lookup for latest Akka announcements",
      "Compose a reply citing the web result"
    ],
    "currentDispatch": {
      "tool": "WEB",
      "subtask": "latest Akka announcements",
      "rationale": "Need to retrieve current news before composing a reply."
    }
  },
  "toolLedger": {
    "entries": [
      {
        "attempt": 1,
        "tool": "WEB",
        "subtask": "latest Akka announcements",
        "verdict": "OK",
        "scrubbedResult": "Akka 3.6.0 released with autonomous agent support... (akka.io, 'Announcements')",
        "blocker": null,
        "recordedAt": "2026-06-28T09:14:05Z"
      }
    ]
  },
  "reply": null,
  "failureReason": null,
  "pauseReason": null,
  "createdAt": "2026-06-28T09:14:00Z",
  "finishedAt": null
}
```

### Session (list form, returned by `GET /api/sessions`)

The list form is `SessionRow` — same fields as `Session`, but `toolLedger.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full session by id on row expand.

### SSE event format

```
event: session-update
data: { "sessionId": "s-9e4…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `sessionId`. The SSE channel also emits `event: control-update` whenever the operator pause flag changes:

```
event: control-update
data: { "paused": true, "reason": "investigating repeated BLOCKED entries", "pausedAt": "2026-06-28T09:15:30Z" }
```
