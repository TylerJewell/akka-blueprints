# API contract — reflection-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "text": String, "submittedBy"?: String }` | `202 { "taskId": String }` | `ReflectionEndpoint` → `SubmissionQueue` |
| `GET` | `/api/tasks` | — | `200 [ Task... ]` (optional `?status=GENERATING\|REFLECTING\|ACCEPTED\|REJECTED_FINAL` — filtered client-side from `getAllTasks`) | `ReflectionEndpoint` ← `TasksView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `ReflectionEndpoint` ← `TasksView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `ReflectionEndpoint` ← `TasksView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/tasks`

- Missing `submittedBy` → `"anonymous"`.
- `text` must be non-empty and at most 4000 characters; otherwise `400`.
- Duplicate-detection window: 10 s on `(text, submittedBy)`; the second submission returns the first `taskId` (200) instead of starting a new workflow.

## JSON shapes

### Task

```json
{
  "taskId": "t-3c9e1…",
  "promptText": "Explain what an event-sourced entity is.",
  "maxIterations": 4,
  "status": "ACCEPTED",
  "iterations": [
    {
      "iterationNumber": 1,
      "response": {
        "text": "An event-sourced entity stores its state as an append-only log…",
        "wordCount": 72,
        "generatedAt": "2026-06-28T10:01:05Z"
      },
      "reflection": {
        "verdict": "REVISE",
        "notes": {
          "changeRequests": [
            "Opening paragraph conflates commands and events; separate them.",
            "Second paragraph does not explain recovery semantics.",
            "Conclusion repeats the opening without adding a takeaway."
          ],
          "overallRationale": "Completeness and coherence fall below threshold."
        },
        "score": 3,
        "reflectedAt": "2026-06-28T10:01:17Z"
      }
    },
    {
      "iterationNumber": 2,
      "response": {
        "text": "An event-sourced entity handles two kinds of interactions separately…",
        "wordCount": 88,
        "generatedAt": "2026-06-28T10:01:25Z"
      },
      "reflection": {
        "verdict": "ACCEPT",
        "notes": {
          "changeRequests": [],
          "overallRationale": "All four dimensions score 4 or above."
        },
        "score": 4,
        "reflectedAt": "2026-06-28T10:01:36Z"
      }
    }
  ],
  "acceptedIterationNumber": 2,
  "acceptedResponse": "An event-sourced entity handles two kinds of interactions separately…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T10:01:02Z",
  "finishedAt": "2026-06-28T10:01:37Z"
}
```

### SSE event format

```
event: task-update
data: { "taskId": "t-3c9e1…", "status": "REFLECTING", "iterations": [...], ... }
```

One event per state transition. Clients reconcile by `taskId`. The full `Task` JSON is included so a fresh client can render the row without a separate fetch.
