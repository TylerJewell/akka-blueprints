# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/task-request` | `{ "request": "string" }` | `{ "taskId": "uuid" }` | TaskEndpoint → ApprovalWorkflow |
| POST | `/api/tasks/{taskId}/approve` | `{ "approvedBy": "string", "comment": "string" }` | `200` \| `404` | TaskEndpoint → TaskEntity |
| POST | `/api/tasks/{taskId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | TaskEndpoint → TaskEntity |
| GET | `/api/tasks` | — | `{ "tasks": [Task, ...] }` | TaskEndpoint → TasksView |
| GET | `/api/tasks/{taskId}` | — | `Task` \| `404` | TaskEndpoint → TaskEntity |
| GET | `/api/tasks/sse` | — | SSE stream of `Task` | TaskEndpoint → TasksView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | TaskEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | TaskEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | TaskEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Task` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "request": "string or null",
  "status": "PROCESSED | APPROVED | REJECTED | FINALIZED",
  "processedAt": "ISO-8601 or null",
  "taskSummary": "string or null",
  "taskRecommendation": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverComment": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "finalizedAt": "ISO-8601 or null",
  "finalizedOutcome": "string or null"
}
```

## SSE event format

`GET /api/tasks/sse` streams the `TasksView` rows. Each message:

```
event: task
data: { ...Task... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
