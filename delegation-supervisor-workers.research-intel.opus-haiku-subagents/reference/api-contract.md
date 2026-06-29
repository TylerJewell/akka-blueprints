# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/analysis` | `{ "question": "string", "images": [ImagePayload, ...] }` | `{ "jobId": "uuid" }` | `AnalysisEndpoint` → `BatchQueue` |
| GET | `/api/analysis` | — | `{ "jobs": [AnalysisJobRow, ...] }` | `AnalysisEndpoint` → `AnalysisView` |
| GET | `/api/analysis?status=SYNTHESISED` | — | filtered list (client-side filter) | `AnalysisEndpoint` |
| GET | `/api/analysis/{id}` | — | `AnalysisJobRow` or 404 | `AnalysisEndpoint` |
| GET | `/api/analysis/sse` | — | `text/event-stream` of `AnalysisJobRow` | `AnalysisEndpoint` → `AnalysisView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AnalysisEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/analysis` request:

```json
{
  "question": "What objects are visible and are there any safety hazards?",
  "images": [
    {
      "imageRef": "img-001",
      "base64Data": "data:image/jpeg;base64,/9j/4AAQ...",
      "mimeType": "image/jpeg"
    },
    {
      "imageRef": "img-002",
      "base64Data": "data:image/png;base64,iVBORw0KGgo...",
      "mimeType": "image/png"
    }
  ]
}
```

`AnalysisJobRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "jobId": "uuid",
  "question": "string",
  "imageRefs": ["img-001", "img-002"],
  "status": "QUEUED | ANALYSING | SYNTHESISED | PARTIAL | BLOCKED",
  "sanitizedImages": [
    { "imageRef": "img-001", "sanitizedBase64Data": "...", "mimeType": "image/jpeg", "piiDetected": false }
  ],
  "reports": [
    {
      "imageRef": "img-001",
      "description": "A warehouse interior with stacked pallets...",
      "tags": ["indoor", "warehouse", "storage"],
      "detectedLabels": [
        { "label": "pallet", "confidence": 0.94 },
        { "label": "forklift", "confidence": 0.87 }
      ],
      "analysedAt": "2026-06-28T14:30:00Z"
    }
  ],
  "synthesised": {
    "answer": "The images show a warehouse environment...",
    "imageSummaries": [ "...per-image summary..." ],
    "guardrailVerdict": "ok",
    "synthesisedAt": "2026-06-28T14:30:05Z"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/analysis/sse` emits one event per job change:

```
event: job
data: { ...AnalysisJobRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `jobId`. No polling.
