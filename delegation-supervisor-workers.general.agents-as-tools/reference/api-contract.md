# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "description": "string" }` | `{ "taskId": "uuid" }` | `TaskEndpoint` → `TaskQueue` |
| GET | `/api/tasks` | — | `{ "tasks": [TaskRecordRow, ...] }` | `TaskEndpoint` → `TaskView` |
| GET | `/api/tasks?status=COMPLETED` | — | filtered list (client-side filter) | `TaskEndpoint` |
| GET | `/api/tasks/{id}` | — | `TaskRecordRow` or 404 | `TaskEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `TaskRecordRow` | `TaskEndpoint` → `TaskView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TaskEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{ "description": "Draft a short memo summarising the key decisions from today's meeting." }
```

`POST /api/tasks` response:

```json
{ "taskId": "uuid-string" }
```

`TaskRecordRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "taskId": "uuid",
  "description": "string",
  "status": "QUEUED | PROCESSING | COMPLETED | FAILED | REJECTED",
  "draft": { "text": "...", "wordCount": 87, "draftedAt": "ISO-8601" },
  "data": {
    "entities": [
      { "name": "...", "kind": "person | organisation | location | concept", "context": "..." }
    ],
    "summarisedAt": "ISO-8601"
  },
  "result": { "answer": "...", "toolsUsed": ["writer_tool"], "assembledAt": "ISO-8601" },
  "rejectionReason": "string or null",
  "qualityScore": "1-5 or null",
  "qualityRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/tasks/sse` emits one event per task record change:

```
event: task
data: { ...TaskRecordRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `taskId`. No polling.
