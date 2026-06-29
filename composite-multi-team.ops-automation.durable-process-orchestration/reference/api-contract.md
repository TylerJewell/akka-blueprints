# API contract — sk-process

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "template": String, "payload"?: String, "submittedBy"?: String }` | `202 { "jobId": String }` | `JobQueueEndpoint` → `JobQueue` |
| `GET` | `/api/jobs` | — | `200 [ Job... ]` | `JobQueueEndpoint` ← `JobBoardView` |
| `GET` | `/api/jobs?status=…` | — | `200 [ Job... ]` (filtered client-side) | `JobQueueEndpoint` ← `JobBoardView` |
| `GET` | `/api/jobs/{id}` | — | `200 Job` / `404` | `JobQueueEndpoint` ← `JobEntity` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `JobQueueEndpoint` ← `JobBoardView` |
| `GET` | `/api/jobs/{id}/steps` | — | `200 [ Step... ]` | `JobQueueEndpoint` ← `StepBoardView` |
| `GET` | `/api/steps/sse` | — | `text/event-stream` (one event per step change) | `JobQueueEndpoint` ← `StepBoardView` |
| `POST` | `/api/jobs/{id}/pause` | — | `200 Job` / `409` (job not `EXECUTING`) / `404` | `JobQueueEndpoint` → `JobEntity`, `ProcessOrchestrationWorkflow` |
| `POST` | `/api/jobs/{id}/resume` | — | `200 Job` / `409` (job not `PAUSED`) / `404` | `JobQueueEndpoint` → `JobEntity`, `ProcessOrchestrationWorkflow` |
| `POST` | `/api/jobs/{id}/audit-review` | `{ "reviewedBy": String, "outcome": "APPROVED"\|"FLAGGED", "notes": String }` | `200 Job` / `409` (job not `COMPLETED`) / `404` | `JobQueueEndpoint` → `JobEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Job

```json
{
  "jobId": "j-4a9…",
  "template": "dependency-audit",
  "payload": null,
  "submittedBy": "operator-1",
  "status": "COMPLETED",
  "stepPlan": {
    "objective": "Audit third-party dependencies for version drift, vulnerabilities, and licence conflicts.",
    "steps": [
      { "name": "Inventory dependencies", "description": "List all declared dependencies.", "expectedOutput": "Structured list of name/version pairs." },
      { "name": "Scan for vulnerabilities", "description": "Cross-reference with CVE dataset.", "expectedOutput": "List of CVE ids and severity ratings." }
    ]
  },
  "detailedPlan": { "approach": "Four independent checks in display order.", "steps": [] },
  "stepIds": ["j-4a9-p0", "j-4a9-p1", "j-4a9-p2", "j-4a9-p3"],
  "completedBatch": { "jobId": "j-4a9", "totalOutputTokens": 284 },
  "qualityVerdict": { "outcome": "PASS", "mustRetry": [] },
  "jobSummary": {
    "overview": "The dependency audit completed successfully. Three vulnerabilities were identified and two licence conflicts require follow-up.",
    "keyOutputs": ["3 CVEs found (1 critical, 1 high, 1 moderate)", "2 licence conflicts flagged"],
    "resultRef": "https://ops.example/results/j-4a9"
  },
  "resultRef": "https://ops.example/results/j-4a9",
  "completedAt": "2026-06-28T10:44:19Z",
  "stepSignals": [
    { "stage": "batch", "score": 84, "flags": [], "evaluatedAt": "2026-06-28T10:43:50Z" },
    { "stage": "validation", "score": 91, "flags": [], "evaluatedAt": "2026-06-28T10:44:10Z" }
  ],
  "auditReview": null,
  "retryCount": 0,
  "createdAt": "2026-06-28T10:41:05Z"
}
```

The `Job` returned by `GET /api/jobs/{id}` carries the full `completedBatch.outputs` with step results; the board rows streamed over SSE drop those heavy fields (see `data-model.md`). Lifecycle fields (`stepPlan`, `detailedPlan`, `completedBatch`, `qualityVerdict`, `jobSummary`, `resultRef`, `completedAt`, `auditReview`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### Step

```json
{
  "stepId": "j-4a9-p1",
  "jobId": "j-4a9",
  "name": "Scan for vulnerabilities",
  "description": "Cross-reference inventoried dependencies against the CVE dataset.",
  "expectedOutput": "A list of CVE ids and severity ratings.",
  "sequence": 2,
  "status": "DONE",
  "claimedBy": "executor-2",
  "claimedAt": "2026-06-28T10:42:30Z",
  "outputTokens": 62,
  "doneAt": "2026-06-28T10:43:01Z",
  "createdAt": "2026-06-28T10:41:40Z"
}
```

The board row carries `outputTokens` and a `resultPresent` flag rather than the full `result`; `GET /api/jobs/{id}/steps` may include the result for the App UI's per-step expand.

### AuditReview (request and recorded form)

```json
{ "reviewedBy": "audit-team-1", "outcome": "FLAGGED", "notes": "Critical CVE should trigger an immediate remediation ticket; summary should reference the ticket id." }
```

Recorded on the job as `auditReview` with an added `reviewedAt`. Accepted only when the job status is `COMPLETED`; otherwise the endpoint returns `409`.

### SSE event format

```
event: job-update
data: { "jobId": "j-4a9", "status": "EXECUTING", ... }

event: step-update
data: { "stepId": "j-4a9-p1", "status": "CLAIMED", "claimedBy": "executor-2", ... }
```

`GET /api/jobs/sse` emits one `job-update` per job state transition; `GET /api/steps/sse` emits one `step-update` per step transition. Clients reconcile jobs by `jobId` and group the step board by `status`. The pause and resume transitions emit a `job-update` with `status=PAUSED` and `status=EXECUTING` respectively so the operator dashboard updates immediately.
