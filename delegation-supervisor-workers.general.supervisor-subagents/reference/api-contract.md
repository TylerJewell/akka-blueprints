# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "description": "string" }` | `{ "taskId": "uuid" }` | `TaskEndpoint` → `TaskQueue` |
| GET | `/api/tasks` | — | `{ "tasks": [TaskRow, ...] }` | `TaskEndpoint` → `TaskView` |
| GET | `/api/tasks?status=COMPLETED` | — | filtered list (client-side filter) | `TaskEndpoint` |
| GET | `/api/tasks/{id}` | — | `TaskRow` or 404 | `TaskEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `TaskRow` | `TaskEndpoint` → `TaskView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TaskEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{ "description": "Summarise the key risks of running LLM inference in a regulated environment." }
```

`TaskRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "taskId": "uuid",
  "description": "string",
  "status": "RECEIVED | ROUTING | IN_PROGRESS | COMPLETED | DEGRADED | BLOCKED",
  "routingPlan": {
    "dataQuery": "string",
    "summaryPrompt": "string"
  },
  "data": {
    "items": [
      { "label": "string", "value": "string", "source": "string" }
    ],
    "retrievedAt": "ISO-8601"
  },
  "summary": {
    "narrative": "string",
    "keyPoints": ["string"],
    "generatedAt": "ISO-8601"
  },
  "result": {
    "narrative": "string",
    "routingRationale": "string",
    "assembledAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/tasks/sse` emits one event per task change:

```
event: task
data: { ...TaskRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `taskId`. No polling.
