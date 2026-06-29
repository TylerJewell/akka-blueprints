# API contract — hierarchical-workflow-automation

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/requests` | `{ "description": String, "requestedBy"?: String }` | `202 { "requestId": String }` | `WorkflowEndpoint` → `RequestQueue` |
| `GET` | `/api/requests` | — | `200 [ WorkflowState... ]` | `WorkflowEndpoint` ← `WorkflowBoardView` |
| `GET` | `/api/requests?status=…` | — | `200 [ WorkflowState... ]` (filtered client-side) | `WorkflowEndpoint` ← `WorkflowBoardView` |
| `GET` | `/api/requests/{id}` | — | `200 WorkflowState` / `404` | `WorkflowEndpoint` ← `WorkflowEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` (one event per workflow change) | `WorkflowEndpoint` ← `WorkflowBoardView` |
| `GET` | `/api/requests/{id}/tasks` | — | `200 [ Task... ]` | `WorkflowEndpoint` ← `TaskBoardView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `WorkflowEndpoint` ← `TaskBoardView` |
| `POST` | `/api/requests/{id}/compliance-review` | `{ "reviewedBy": String, "outcome": "CLEARED"\|"FLAGGED", "comments": String }` | `200 WorkflowState` / `409` (request not `COMPLETED`) / `404` | `WorkflowEndpoint` → `WorkflowEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### WorkflowState

```json
{
  "requestId": "req-4d7…",
  "description": "Rotate TLS certificates across the API gateway cluster",
  "requestedBy": "ui",
  "status": "COMPLETED",
  "plan": {
    "objective": "Rotate expired TLS certificates on all API gateway nodes and verify the cluster is serving with the new certificates.",
    "targetSystems": ["api-gateway-cluster", "certificate-authority"],
    "taskDescriptions": ["Identify nodes with expiring certificates", "Request new certificates", "Deploy and verify on each node"]
  },
  "discoverySummary": {
    "overview": "Two of the three API gateway nodes carry certificates expiring within three days; the CA is reachable.",
    "keyFindings": ["node-01 certificate expires in 3 days", "node-02 certificate already expired", "ca-server-primary reachable on port 8443"]
  },
  "taskIds": ["req-4d7-t0", "req-4d7-t1", "req-4d7-t2"],
  "executionSummary": { "headline": "TLS rotation complete", "tasksCompleted": 3 },
  "validationVerdict": { "outcome": "PASS", "mustRetry": [] },
  "report": { "title": "TLS Certificate Rotation — Complete", "preparedBy": "Workflow Automation" },
  "reportUrl": "https://ops.example/req-4d7",
  "completedAt": "2026-06-28T09:22:15Z",
  "auditEvals": [
    { "stage": "discovery", "score": 91, "flags": [], "evaluatedAt": "2026-06-28T09:19:40Z" },
    { "stage": "execution", "score": 85, "flags": [], "evaluatedAt": "2026-06-28T09:21:30Z" },
    { "stage": "validation", "score": 95, "flags": [], "evaluatedAt": "2026-06-28T09:22:00Z" }
  ],
  "complianceReview": null,
  "retryCount": 0,
  "createdAt": "2026-06-28T09:18:50Z"
}
```

The `WorkflowState` returned by `GET /api/requests/{id}` carries the full `report.body` and `executionSummary.outcomes`; the board rows streamed over SSE drop those heavy fields (see `data-model.md`). Lifecycle fields (`plan`, `discoverySummary`, `executionSummary`, `validationVerdict`, `report`, `reportUrl`, `completedAt`, `complianceReview`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### Task

```json
{
  "taskId": "req-4d7-t1",
  "requestId": "req-4d7",
  "title": "Deploy certificate to node-01",
  "instruction": "Install the new certificate on api-gateway-node-01, reload the TLS listener, and verify the node returns the new certificate on port 443.",
  "status": "DONE",
  "claimedBy": "executor-1",
  "claimedAt": "2026-06-28T09:20:45Z",
  "stepsCompleted": 3,
  "completedAt": "2026-06-28T09:21:10Z",
  "createdAt": "2026-06-28T09:20:30Z"
}
```

The board row carries `stepsCompleted` and a `resultPresent` boolean rather than the full `result`; `GET /api/requests/{id}/tasks` may include the result for the App UI's per-task expand.

### ComplianceReview (request and recorded form)

```json
{ "reviewedBy": "ops-compliance-1", "outcome": "FLAGGED", "comments": "Certificate deployed to node-02 lacked a change-ticket reference." }
```

Recorded on the workflow as `complianceReview` with an added `reviewedAt`. Accepted only when the workflow status is `COMPLETED`; otherwise the endpoint returns `409`.

### SSE event format

```
event: workflow-update
data: { "requestId": "req-4d7", "status": "VALIDATING", ... }

event: task-update
data: { "taskId": "req-4d7-t1", "status": "DONE", ... }
```

`GET /api/requests/sse` emits one `workflow-update` per workflow state transition; `GET /api/tasks/sse` emits one `task-update` per task transition. Clients reconcile requests by `requestId` and group the task board by `status`.
