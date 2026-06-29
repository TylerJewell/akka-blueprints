# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "payload": "string" }` | `{ "jobId": "uuid" }` | `TaskEndpoint` → `JobQueue` |
| GET | `/api/tasks` | — | `{ "jobs": [TaskJobRow, ...] }` | `TaskEndpoint` → `TaskJobView` |
| GET | `/api/tasks?status=CONSOLIDATED` | — | filtered list (client-side filter) | `TaskEndpoint` |
| GET | `/api/tasks/{id}` | — | `TaskJobRow` or 404 | `TaskEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `TaskJobRow` | `TaskEndpoint` → `TaskJobView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TaskEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{ "payload": "Analyze the dependency graph of a distributed caching layer" }
```

`TaskJobRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "jobId": "uuid",
  "payload": "string",
  "status": "QUEUED | PROCESSING | CONSOLIDATED | DEGRADED | REJECTED",
  "structural": {
    "elements": [
      { "label": "string", "category": "component | dependency | interface | constraint | resource", "detail": "string" }
    ],
    "processedAt": "ISO-8601"
  },
  "contextual": {
    "enrichedContext": "string",
    "tags": ["string"],
    "enrichedAt": "ISO-8601"
  },
  "consolidated": {
    "summary": "string",
    "validationVerdict": "ok",
    "consolidatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "qualityScore": "1-5 or null",
  "qualityRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/tasks/sse` emits one event per job change:

```
event: job
data: { ...TaskJobRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `jobId`. No polling.
