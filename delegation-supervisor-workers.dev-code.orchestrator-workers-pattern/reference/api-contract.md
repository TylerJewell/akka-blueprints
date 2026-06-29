# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "description": "string", "targetFiles": ["string"] }` | `{ "taskId": "uuid" }` | `EditEndpoint` → `EditJobQueue` |
| GET | `/api/tasks` | — | `{ "tasks": [EditTaskRow, ...] }` | `EditEndpoint` → `EditTaskView` |
| GET | `/api/tasks?status=COMPLETED` | — | filtered list (client-side filter) | `EditEndpoint` |
| GET | `/api/tasks/{id}` | — | `EditTaskRow` or 404 | `EditEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `EditTaskRow` | `EditEndpoint` → `EditTaskView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `EditEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `EditEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `EditEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{
  "description": "Rename the processData method to transformPayload across all usages",
  "targetFiles": ["src/DataProcessor.java", "src/PipelineRunner.java"]
}
```

`EditTaskRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "taskId": "uuid",
  "description": "string",
  "targetFiles": ["string"],
  "status": "PLANNING | IN_PROGRESS | COMPLETED | DEGRADED | BLOCKED",
  "plan": {
    "instructions": [
      { "filePath": "src/DataProcessor.java", "instruction": "Rename method processData to transformPayload" }
    ],
    "objective": "string"
  },
  "editedFiles": [
    { "filePath": "string", "editedContent": "string", "diffSummary": "string" }
  ],
  "reviews": [
    { "filePath": "string", "approved": true, "feedback": "string" }
  ],
  "changeset": {
    "summary": "string",
    "qualityVerdict": "ok | needs-review",
    "completedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "qualityScore": "1-5 or null",
  "qualityRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/tasks/sse` emits one event per task change:

```
event: task
data: { ...EditTaskRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `taskId`. No polling.
