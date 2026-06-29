# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "description": "string" }` | `{ "taskId": "uuid" }` | `CollaborationEndpoint` → `TaskRequestQueue` |
| GET | `/api/tasks` | — | `{ "tasks": [CollaborationTaskRow, ...] }` | `CollaborationEndpoint` → `TaskView` |
| GET | `/api/tasks?status=COMPLETED` | — | filtered list (client-side filter) | `CollaborationEndpoint` |
| GET | `/api/tasks/{id}` | — | `CollaborationTaskRow` or 404 | `CollaborationEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `CollaborationTaskRow` | `CollaborationEndpoint` → `TaskView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `CollaborationEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `CollaborationEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `CollaborationEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{ "description": "Compare global renewable energy adoption rates and show trends by region over the last 5 years." }
```

`CollaborationTaskRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "taskId": "uuid",
  "description": "string",
  "status": "ROUTING | IN_PROGRESS | COMPLETED | DEGRADED | BLOCKED",
  "routingDecision": {
    "researchQuery": "string",
    "chartDescription": "string"
  },
  "researchOutput": {
    "findings": [
      { "headline": "...", "source": "...", "content": "..." }
    ],
    "gatheredAt": "ISO-8601"
  },
  "chartData": {
    "chartType": "bar | line",
    "title": "string",
    "series": [
      { "label": "string", "values": [1.2, 3.4, 5.6] }
    ],
    "generatedAt": "ISO-8601"
  },
  "result": {
    "summary": "string",
    "guardrailVerdict": "ok",
    "completedAt": "ISO-8601"
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
data: { ...CollaborationTaskRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `taskId`. No polling.
