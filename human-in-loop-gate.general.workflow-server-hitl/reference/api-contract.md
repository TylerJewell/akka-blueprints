# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/jobs` | `{ "requestPayload": "string" }` | `{ "jobId": "uuid" }` | WorkflowEndpoint → JobWorkflow |
| POST | `/api/jobs/{jobId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | WorkflowEndpoint → JobEntity |
| POST | `/api/jobs/{jobId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | WorkflowEndpoint → JobEntity |
| GET | `/api/jobs` | — | `{ "jobs": [Job, ...] }` | WorkflowEndpoint → JobsView |
| GET | `/api/jobs/{jobId}` | — | `Job` \| `404` | WorkflowEndpoint → JobEntity |
| GET | `/api/jobs/sse` | — | SSE stream of `Job` | WorkflowEndpoint → JobsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | WorkflowEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | WorkflowEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | WorkflowEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Job` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "requestPayload": "string or null",
  "status": "ANALYZED | APPROVED | REJECTED | COMPLETED",
  "analyzedAt": "ISO-8601 or null",
  "findings": "string or null",
  "riskLevel": "LOW | MEDIUM | HIGH | null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "completedAt": "ISO-8601 or null",
  "output": "string or null"
}
```

## SSE event format

`GET /api/jobs/sse` streams the `JobsView` rows. Each message:

```
event: job
data: { ...Job... }
```

The UI subscribes once on load and replaces the matching row by `id` on each event. No polling is needed; the stream carries all state transitions as they occur.
