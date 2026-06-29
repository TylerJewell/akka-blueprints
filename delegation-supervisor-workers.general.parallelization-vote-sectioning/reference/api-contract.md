# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/jobs` | `{ "prompt": "string", "mode": "SECTIONING\|VOTING" }` | `{ "jobId": "uuid" }` | `JobEndpoint` → `JobQueue` |
| GET | `/api/jobs` | — | `{ "jobs": [JobRow, ...] }` | `JobEndpoint` → `JobView` |
| GET | `/api/jobs?status=AGGREGATED` | — | filtered list (client-side filter) | `JobEndpoint` |
| GET | `/api/jobs/{id}` | — | `JobRow` or 404 | `JobEndpoint` |
| GET | `/api/jobs/sse` | — | `text/event-stream` of `JobRow` | `JobEndpoint` → `JobView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `JobEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `JobEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `JobEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/jobs` request:

```json
{ "prompt": "What are the main arguments for and against ranked-choice voting?", "mode": "VOTING" }
```

`JobRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "jobId": "uuid",
  "prompt": "string",
  "mode": "SECTIONING | VOTING",
  "status": "PARTITIONING | IN_PROGRESS | AGGREGATED | PARTIAL | BLOCKED",
  "workerCount": 3,
  "sectionResults": [
    { "sectionIndex": 0, "content": "...", "processedAt": "ISO-8601" }
  ],
  "voteResults": [
    { "workerIndex": 0, "answer": "...", "confidence": 0.82, "votedAt": "ISO-8601" }
  ],
  "aggregated": {
    "answer": "...",
    "mode": "VOTING",
    "sectionOutputs": [],
    "voteOutputs": [],
    "aggregationVerdict": "ok",
    "aggregatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/jobs/sse` emits one event per job change:

```
event: job
data: { ...JobRow... }

```

The App UI tab subscribes via `EventSource` and upserts each row into its live list by `jobId`. No polling.
