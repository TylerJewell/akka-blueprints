# API contract — todoist-organizer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/organizer` | — | `200 [ OrganizerTask... ]` (sorted newest-first) | `OrganizerEndpoint` ← `OrganizerView` |
| `GET` | `/api/organizer/{id}` | — | `200 OrganizerTask` / `404` | `OrganizerEndpoint` ← `OrganizerView` |
| `GET` | `/api/organizer/sse` | — | `text/event-stream` | `OrganizerEndpoint` ← `OrganizerView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### OrganizerTask

```json
{
  "taskId": "td-8a2f…",
  "original": {
    "taskId": "td-8a2f…",
    "content": "Write Q3 OKR review doc",
    "description": "",
    "projectId": null,
    "labels": [],
    "priority": "p4",
    "fetchedAt": "2026-06-28T09:00:00Z"
  },
  "classification": {
    "targetProjectId": "work-proj-001",
    "targetProjectName": "Work",
    "targetLabels": ["writing", "okr"],
    "priorityLevel": "p2",
    "confidence": "high",
    "reason": "OKR review is a core work deliverable."
  },
  "guardrailVerdict": {
    "allowed": true,
    "reason": "Confidence high; project in allow-list."
  },
  "update": {
    "targetProjectId": "work-proj-001",
    "targetProjectName": "Work",
    "appliedLabels": ["writing", "okr"],
    "priorityLevel": "p2",
    "updatedAt": "2026-06-28T09:00:04Z"
  },
  "evalScore": 5,
  "evalRationale": "Project and labels are an unambiguous match for the task content.",
  "status": "UPDATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:04Z"
}
```

### SSE event format

```
event: organizer-update
data: { "taskId": "...", "status": "UPDATED", ... }
```

One event per state transition. Clients reconcile by `taskId`.
