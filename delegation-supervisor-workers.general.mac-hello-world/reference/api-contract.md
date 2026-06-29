# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/greetings` | `{ "name": "string", "context": "string" }` | `{ "requestId": "uuid" }` | `GreetingEndpoint` → `RequestQueue` |
| GET | `/api/greetings` | — | `{ "requests": [GreetingRequestRow, ...] }` | `GreetingEndpoint` → `GreetingView` |
| GET | `/api/greetings?status=COMPLETED` | — | filtered list (client-side filter) | `GreetingEndpoint` |
| GET | `/api/greetings/{id}` | — | `GreetingRequestRow` or 404 | `GreetingEndpoint` |
| GET | `/api/greetings/sse` | — | `text/event-stream` of `GreetingRequestRow` | `GreetingEndpoint` → `GreetingView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `GreetingEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `GreetingEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `GreetingEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/greetings` request:

```json
{ "name": "Amara", "context": "welcoming a new team member on their first day" }
```

`GreetingRequestRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "requestId": "uuid",
  "name": "string",
  "context": "string",
  "status": "PENDING | IN_PROGRESS | COMPLETED | DEGRADED",
  "draft": { "message": "...", "draftedAt": "ISO-8601" },
  "followUp": { "action": "...", "advisedAt": "ISO-8601" },
  "response": { "message": "...", "followUpAction": "...", "composedAt": "ISO-8601" },
  "failureReason": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/greetings/sse` emits one event per request change:

```
event: greeting
data: { ...GreetingRequestRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `requestId`. No polling.
