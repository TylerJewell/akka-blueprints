# API contract — activity-interrupt-cancellation

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/activities` | — | `200 [ ActivityRecord... ]` (sorted newest-first) | `ActivityEndpoint` ← `ActivityView` |
| `GET` | `/api/activities/{id}` | — | `200 ActivityRecord` / `404` | `ActivityEndpoint` ← `ActivityView` |
| `POST` | `/api/activities` | `{ "name": String, "description": String, "kind": TaskKind, "estimatedSteps": int }` | `201 ActivityRecord` (status QUEUED) | `ActivityEndpoint` → `TaskQueue` → `ActivityEntity` |
| `POST` | `/api/activities/{id}/cancel` | `{ "requestedBy": String, "reason": String }` | `200 ActivityRecord` (status CANCELLING) | `ActivityEndpoint` → `ActivityEntity` |
| `GET` | `/api/activities/sse` | — | `text/event-stream` | `ActivityEndpoint` ← `ActivityView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### ActivityRecord

```json
{
  "taskId": "act-7a3f…",
  "task": {
    "taskId": "act-7a3f…",
    "name": "Provision dev VPC",
    "description": "Create VPC, subnets, security groups for dev environment.",
    "kind": "INFRA_PROVISION",
    "estimatedSteps": 5,
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "stepLog": [
    {
      "stepIndex": 1,
      "summary": "Created VPC vpc-00000001 in us-east-1.",
      "terminal": false,
      "partialArtifact": "vpc-00000001",
      "completedAt": "2026-06-28T10:00:12Z"
    },
    {
      "stepIndex": 2,
      "summary": "Cancelled at operator request — no further steps executed.",
      "terminal": true,
      "partialArtifact": null,
      "completedAt": "2026-06-28T10:00:28Z"
    }
  ],
  "cleanupPlan": {
    "steps": [
      "Delete VPC vpc-00000001 and its associated subnets.",
      "Verify no ENIs remain attached before deletion."
    ],
    "reason": "VPC was created but subnet and security group configuration did not complete.",
    "rollbackNote": null,
    "planCreatedAt": "2026-06-28T10:00:31Z"
  },
  "cancellationRequest": {
    "requestedBy": "ops-eng-42",
    "reason": "Wrong region specified.",
    "requestedAt": "2026-06-28T10:00:25Z"
  },
  "status": "CANCELLED",
  "createdAt": "2026-06-28T10:00:00Z",
  "startedAt": "2026-06-28T10:00:05Z",
  "finishedAt": "2026-06-28T10:00:31Z"
}
```

### POST /api/activities body

```json
{
  "name": "Generate monthly cost report",
  "description": "Pull spend data from all accounts and produce a CSV summary.",
  "kind": "REPORT_GENERATE",
  "estimatedSteps": 4
}
```

### POST /api/activities/{id}/cancel body

```json
{
  "requestedBy": "ops-eng-42",
  "reason": "Wrong region specified."
}
```

### SSE event format

```
event: activity-update
data: { "taskId": "act-7a3f…", "status": "CANCELLING", ... }
```

One event per state transition and per `StepCompleted`. Clients reconcile by `taskId`.
