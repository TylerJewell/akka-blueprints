# API contract — project-tracker

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/board` | — | `200 [ PlannerTask... ]` (sorted overdue-first) | `BoardEndpoint` ← `BoardView` |
| `GET` | `/api/board/{id}` | — | `200 PlannerTask` / `404` | `BoardEndpoint` ← `BoardView` |
| `POST` | `/api/board/{id}/complete` | `{ "resolvedBy": String }` | `200 PlannerTask` (status now COMPLETED) | `BoardEndpoint` → `PlannerTaskEntity` |
| `POST` | `/api/board/{id}/escalate` | `{ "escalatedBy": String, "reason": String }` | `200 PlannerTask` (status now ESCALATED) | `BoardEndpoint` → `PlannerTaskEntity` |
| `GET` | `/api/board/sse` | — | `text/event-stream` | `BoardEndpoint` ← `BoardView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### PlannerTask

```json
{
  "taskId": "task-4a9f…",
  "summary": {
    "taskId": "task-4a9f…",
    "title": "Migrate auth service to OAuth 2.1",
    "description": "Update token exchange flow per RFC 9207…",
    "dueDate": "2026-07-04T17:00:00Z",
    "priority": "HIGH"
  },
  "assignment": {
    "recommendedOwnerId": "eng-04",
    "recommendedOwnerName": "Dana Reyes",
    "rationale": "Has prior experience with auth service internals.",
    "confidence": "high"
  },
  "currentOwnerId": "eng-04",
  "currentOwnerName": "Dana Reyes",
  "nudgeCount": 0,
  "lastNudgedAt": null,
  "evalScore": null,
  "evalRationale": null,
  "status": "IN_PROGRESS",
  "createdAt": "2026-06-28T09:00:00Z",
  "completedAt": null
}
```

### DispatchResult (internal — logged on blocked actions)

```json
{
  "actionId": "g1-check-task-4a9f-assign",
  "toolName": "assignTask",
  "dispatched": false,
  "blockReason": "Owner eng-04_FULL is at capacity; assignment blocked.",
  "attemptedAt": "2026-06-28T09:00:05Z"
}
```

### SSE event format

```
event: board-update
data: { "taskId": "task-4a9f…", "status": "ASSIGNED", "currentOwnerName": "Dana Reyes", … }
```

One event per state transition. Clients reconcile by `taskId`.
