# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "taskDescription": "string" }` | `{ "requestId": "uuid" }` | `TaskEndpoint` → `TaskQueue` |
| GET | `/api/tasks` | — | `{ "tasks": [TaskRequestRow, ...] }` | `TaskEndpoint` → `TaskView` |
| GET | `/api/tasks?status=COMPLETED` | — | filtered list (client-side filter) | `TaskEndpoint` |
| GET | `/api/tasks/{id}` | — | `TaskRequestRow` or 404 | `TaskEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `TaskRequestRow` | `TaskEndpoint` → `TaskView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TaskEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TaskEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{ "taskDescription": "Write a summary of the key tradeoffs in distributed consensus protocols and evaluate which is most suitable for low-latency applications." }
```

`TaskRequestRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "requestId": "uuid",
  "taskDescription": "string",
  "status": "PLANNING | IN_PROGRESS | COMPLETED | DEGRADED | BLOCKED",
  "plan": {
    "planId": "uuid",
    "objective": "string",
    "subTasks": [
      { "subTaskId": "uuid", "workerRole": "WriterWorker", "instruction": "..." },
      { "subTaskId": "uuid", "workerRole": "AnalyzerWorker", "instruction": "..." }
    ]
  },
  "writerOutput": {
    "content": "string",
    "sections": ["Section 1", "Section 2"],
    "producedAt": "ISO-8601"
  },
  "analyzerOutput": {
    "evaluation": "string",
    "findings": ["finding 1", "finding 2"],
    "producedAt": "ISO-8601"
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
data: { ...TaskRequestRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `requestId`. No polling.
