# API contract — sandboxed-code-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/executions` | `SubmitExecutionRequest` | `201 { executionId }` | `ExecutionEndpoint` → `ExecutionEntity` |
| `GET` | `/api/executions` | — | `200 [ Execution... ]` (newest-first) | `ExecutionEndpoint` ← `ExecutionView` |
| `GET` | `/api/executions/{id}` | — | `200 Execution` / `404` | `ExecutionEndpoint` ← `ExecutionView` |
| `GET` | `/api/executions/sse` | — | `text/event-stream` | `ExecutionEndpoint` ← `ExecutionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitExecutionRequest (request body)

```json
{
  "taskDescription": "Compute the mean and standard deviation of [1, 4, 9, 16, 25] and print both values.",
  "backend": "DOCKER",
  "wallClockBudgetSeconds": 30,
  "cpuBudgetSeconds": 10,
  "submittedBy": "dev-user-42"
}
```

`backend` accepts `DOCKER` (default), `E2B`, `MODAL`, or `PLAYWRIGHT`. `wallClockBudgetSeconds` and `cpuBudgetSeconds` default to 30 and 10 if omitted.

### Execution (response body)

```json
{
  "executionId": "e-7f3...",
  "request": {
    "executionId": "e-7f3...",
    "taskDescription": "Compute the mean and standard deviation of [1, 4, 9, 16, 25] and print both values.",
    "backend": "DOCKER",
    "wallClockBudgetSeconds": 30,
    "cpuBudgetSeconds": 10,
    "submittedBy": "dev-user-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "code": {
    "pythonSource": "import statistics\ndata = [1, 4, 9, 16, 25]\nprint(f'Mean: {statistics.mean(data):.2f}')\nprint(f'Std dev: {statistics.stdev(data):.2f}')",
    "codeHash": "a3f9...d1"
  },
  "screening": {
    "decision": "APPROVED",
    "violatedPattern": null,
    "rationale": "No forbidden patterns detected."
  },
  "output": {
    "exitCode": 0,
    "stdout": "Mean: 11.00\nStd dev: 8.35",
    "stderr": "",
    "generatedFiles": [],
    "wallClockMs": 1240,
    "cpuMs": 210
  },
  "haltReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:02Z"
}
```

A halted execution:

```json
{
  "executionId": "e-4a1...",
  "status": "HALTED",
  "haltReason": {
    "limitBreached": "wall-clock",
    "measuredMs": 30012,
    "budgetMs": 30000
  },
  "output": null,
  "finishedAt": "2026-06-28T09:01:30Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: execution-update
data: { "executionId": "e-7f3...", "status": "COMPLETED", "output": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SCREENING`, `APPROVED`, `RUNNING`, `COMPLETED`, `HALTED`, `FAILED`). Clients reconcile by `executionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay history.

## Authorization

ACL: open to the local interface in dev mode. A deployer adding identity wraps the endpoint with their auth middleware and sets `submittedBy` from the authenticated principal rather than the request body. The sandbox dispatchers receive only the code string — no session credentials, no host env vars — regardless of the authorization layer above.
