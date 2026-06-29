# API contract — task-insight-memory

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "taskType": String, "description": String, "acceptanceCriteria"?: String, "requestedBy"?: String }` | `202 { "taskId": String }` | `MemoryEndpoint` → `TaskQueue` |
| `GET` | `/api/tasks` | — | `200 [ TaskRecord... ]` (optional `?status=PENDING\|EXECUTING\|EVALUATED\|VERIFIED\|FAILED` — filtered client-side from `getAllTasks`) | `MemoryEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 TaskRecord` / `404` | `MemoryEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `MemoryEndpoint` ← `TaskView` |
| `GET` | `/api/memory` | — | `200 [ MemoryInsight... ]` (optional `?taskType=<string>`) | `MemoryEndpoint` ← `MemoryView` |
| `GET` | `/api/memory/{insightId}` | — | `200 MemoryInsight` / `404` | `MemoryEndpoint` ← `MemoryView` |
| `POST` | `/api/memory/correction` | `{ "taskType": String, "correctedText": String, "supersedes"?: String }` | `201 { "insightId": String }` | `MemoryEndpoint` → `MemoryEntity` |
| `POST` | `/api/memory/demonstration` | `{ "taskType": String, "demonstrationText": String }` | `201 { "insightId": String }` | `MemoryEndpoint` → `MemoryEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/tasks`

- Missing `acceptanceCriteria` → `"answer is accurate and complete"`.
- Missing `requestedBy` → `"anonymous"`.
- `taskType` must be non-empty and match `[a-z][a-z0-9-]{0,63}`; otherwise `400`.
- `description` must be between 10 and 4000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(taskType, description)`; the second submission returns the first `taskId` (200) instead of starting a new workflow.

## JSON shapes

### TaskRecord

```json
{
  "taskId": "t-9b3c1…",
  "taskType": "data-extraction",
  "description": "Extract the main numerical statistics from the paragraph.",
  "acceptanceCriteria": "All numerical values with their units and direction.",
  "status": "VERIFIED",
  "retrievedInsights": [
    {
      "insightId": "i-4a2f…",
      "taskType": "data-extraction",
      "text": "Paragraphs in this domain mix percentage and absolute figures; extract both forms.",
      "confidence": 0.88,
      "provenance": "VERIFIED_EXPERIENCE"
    }
  ],
  "result": {
    "answer": "Revenue: +14% YoY, $2.3B. Headcount: 4,200 → 4,850.",
    "confidence": 0.92,
    "keyFindings": [
      "Paragraphs in this domain typically mix percentage and absolute figures; extract both forms.",
      "Headcount figures appear as 'from X to Y' patterns; capture both endpoints."
    ],
    "completedAt": "2026-06-28T09:14:03Z"
  },
  "evalVerdict": {
    "outcome": "VERIFIED",
    "notes": {
      "bullets": [],
      "overallRationale": "Both statistics extracted correctly; key findings capture transferable patterns."
    },
    "qualityScore": 0.92,
    "evaluatedAt": "2026-06-28T09:14:11Z"
  },
  "failureReason": null,
  "createdAt": "2026-06-28T09:13:58Z",
  "finishedAt": "2026-06-28T09:14:15Z"
}
```

### MemoryInsight

```json
{
  "insightId": "i-4a2f…",
  "taskType": "data-extraction",
  "text": "Paragraphs in this domain typically mix percentage and absolute figures; extract both forms.",
  "confidence": 0.92,
  "provenance": "VERIFIED_EXPERIENCE",
  "persistedAt": "2026-06-28T09:14:15Z",
  "supersededAt": null
}
```

### Correction body

```json
{
  "taskType": "data-extraction",
  "correctedText": "Always check for implied units; revenue figures may omit the currency symbol.",
  "supersedes": "i-4a2f…"
}
```

When `supersedes` is provided and refers to an existing non-superseded insight, the old insight receives `InsightSuperseded` and the new one is created with `provenance = CORRECTION`.

### SSE event format

```
event: task-update
data: { "taskId": "t-9b3c1…", "status": "VERIFIED", "result": {...}, "evalVerdict": {...}, ... }
```

One event per state transition. Clients reconcile by `taskId`. The full `TaskRecord` JSON is included so a fresh client can render the row without a separate fetch.

### DriftAssessmentRecorded payload (inside MemoryInsight events)

```json
{
  "concentrationRatio": 0.83,
  "avgConfidence": 0.74,
  "dominantTaskType": "summarization",
  "triggeredThreshold": "TYPE_CONCENTRATION",
  "assessedAt": "2026-06-28T09:20:00Z"
}
```

`triggeredThreshold` is one of `TYPE_CONCENTRATION`, `CONFIDENCE_FLOOR`, `BOTH`.
