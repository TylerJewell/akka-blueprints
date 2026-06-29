# API contract — trajectory-eval

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/evaluations` | `{ "scenarioId": String, "taskDescription"?: String, "requestedBy"?: String }` | `202 { "evaluationId": String }` / `404` if no reference path | `TrajectoryEndpoint` → `EvaluationEntity` |
| `GET` | `/api/evaluations` | — | `200 [ Evaluation... ]` (optional `?status=RUNNING\|EVALUATING\|PASSED\|FAILED_FINAL` — filtered client-side from `getAllEvaluations`) | `TrajectoryEndpoint` ← `EvaluationsView` |
| `GET` | `/api/evaluations/{id}` | — | `200 Evaluation` / `404` | `TrajectoryEndpoint` ← `EvaluationsView` |
| `GET` | `/api/evaluations/sse` | — | `text/event-stream` (one event per evaluation change) | `TrajectoryEndpoint` ← `EvaluationsView` |
| `POST` | `/api/references` | `{ "scenarioId": String, "steps": [ ToolCall... ] }` | `200 { "scenarioId": String }` | `TrajectoryEndpoint` → `ReferenceStore` |
| `GET` | `/api/references/{scenarioId}` | — | `200 ReferencePath` / `404` | `TrajectoryEndpoint` ← `ReferenceStore` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/evaluations`

- Missing `taskDescription` → resolved from the stored scenario's own description field.
- Missing `requestedBy` → `"anonymous"`.
- `scenarioId` must have a registered reference path in `ReferenceStore`; otherwise `404`.
- Duplicate-detection window: 10 s on `(scenarioId, requestedBy)`; the second submission returns the first `evaluationId` (200) instead of starting a new workflow.

## JSON shapes

### Evaluation

```json
{
  "evaluationId": "ev-3a1b9…",
  "scenarioId": "kb-refund-routing",
  "taskDescription": "Route a refund request through the knowledge base and ticketing system.",
  "maxAttempts": 4,
  "status": "PASSED",
  "attempts": [
    {
      "attemptNumber": 1,
      "trajectory": {
        "steps": [
          { "toolName": "search_kb", "inputs": {"query": "refund policy"}, "output": "Returns within 30 days.", "calledAt": "2026-06-28T10:01:01Z" },
          { "toolName": "classify_intent", "inputs": {"text": "I want a refund"}, "output": "REFUND_REQUEST", "calledAt": "2026-06-28T10:01:02Z" },
          { "toolName": "route_ticket", "inputs": {"intent": "REFUND_REQUEST"}, "output": "TK-4421", "calledAt": "2026-06-28T10:01:03Z" }
        ],
        "stepCount": 3,
        "recordedAt": "2026-06-28T10:01:03Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "verdict": {
        "verdict": "FAIL",
        "report": {
          "deviations": [
            { "stepIndex": 1, "expectedTool": "enrich_customer", "actualTool": "classify_intent", "note": "Step 1 expected enrich_customer." }
          ],
          "overallRationale": "One step diverges; enrich_customer was skipped."
        },
        "deviationCount": 1,
        "evaluatedAt": "2026-06-28T10:01:10Z"
      }
    },
    {
      "attemptNumber": 2,
      "trajectory": {
        "steps": [
          { "toolName": "search_kb", "inputs": {"query": "refund policy"}, "output": "Returns within 30 days.", "calledAt": "2026-06-28T10:01:20Z" },
          { "toolName": "enrich_customer", "inputs": {"orderId": "ORD-9912"}, "output": "tier: gold", "calledAt": "2026-06-28T10:01:21Z" },
          { "toolName": "route_ticket", "inputs": {"intent": "REFUND_REQUEST", "priority": "high"}, "output": "TK-4422", "calledAt": "2026-06-28T10:01:22Z" }
        ],
        "stepCount": 3,
        "recordedAt": "2026-06-28T10:01:22Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "verdict": {
        "verdict": "PASS",
        "report": { "deviations": [], "overallRationale": "All 3 steps match the reference path." },
        "deviationCount": 0,
        "evaluatedAt": "2026-06-28T10:01:29Z"
      }
    }
  ],
  "passedAttemptNumber": 2,
  "passedTrajectory": { "steps": [...], "stepCount": 3, "recordedAt": "2026-06-28T10:01:22Z" },
  "failureReason": null,
  "createdAt": "2026-06-28T10:00:58Z",
  "finishedAt": "2026-06-28T10:01:30Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",              "detail": "" }
{ "passed": false, "reasonCode": "OVER_STEP_LIMIT",  "detail": "Trajectory has 23 steps; maximum is 20." }
```

`reasonCode` is one of `OK`, `OVER_STEP_LIMIT`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### ReferencePath

```json
{
  "scenarioId": "kb-refund-routing",
  "steps": [
    { "toolName": "search_kb", "inputs": {"query": "refund policy"}, "output": "", "calledAt": "0001-01-01T00:00:00Z" },
    { "toolName": "enrich_customer", "inputs": {}, "output": "", "calledAt": "0001-01-01T00:00:00Z" },
    { "toolName": "route_ticket", "inputs": {}, "output": "", "calledAt": "0001-01-01T00:00:00Z" }
  ],
  "registeredAt": "2026-06-01T09:00:00Z",
  "updatedAt": null
}
```

Note: reference path `ToolCall` records use placeholder `inputs` and `output`; only `toolName` is compared during evaluation.

### SSE event format

```
event: evaluation-update
data: { "evaluationId": "ev-3a1b9…", "status": "EVALUATING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `evaluationId`. The full `Evaluation` JSON is included so a fresh client can render the row without a separate fetch.
