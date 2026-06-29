# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tasks` | `{ "inputText": "string", "operation": "string" }` | `{ "requestId": "uuid" }` | `CompositionEndpoint` → `TaskRequestQueue` |
| GET | `/api/tasks` | — | `{ "requests": [TaskRequestRow, ...] }` | `CompositionEndpoint` → `CompositionView` |
| GET | `/api/tasks?status=COMPLETED` | — | filtered list (client-side filter) | `CompositionEndpoint` |
| GET | `/api/tasks/{id}` | — | `TaskRequestRow` or 404 | `CompositionEndpoint` |
| GET | `/api/tasks/sse` | — | `text/event-stream` of `TaskRequestRow` | `CompositionEndpoint` → `CompositionView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `CompositionEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `CompositionEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `CompositionEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/tasks` request:

```json
{ "inputText": "The EU AI Act establishes risk tiers for AI systems deployed in the European Union.", "operation": "summarize" }
```

`TaskRequestRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "requestId": "uuid",
  "inputText": "string",
  "operation": "string",
  "status": "QUEUED | IN_PROGRESS | COMPLETED | BLOCKED | TIMED_OUT",
  "routing": { "selectedTool": "summarize | classify | translate", "reasoning": "string" },
  "toolResult": { "toolName": "string", "output": "string", "returnedAt": "ISO-8601" },
  "result": { "operation": "string", "toolUsed": "string", "output": "string", "guardrailVerdict": "ok", "completedAt": "ISO-8601" },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/tasks/sse` emits one event per request state change:

```
event: task-request
data: { ...TaskRequestRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `requestId`. No polling.
