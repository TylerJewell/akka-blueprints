# API contract — multi-agent-eval-harness

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `{ "suiteName": String, "passThreshold"?: Double, "requestedBy"?: String }` | `202 { "runId": String }` | `EvalEndpoint` → `ScenarioRegistry` |
| `GET` | `/api/runs` | — | `200 [ EvalRun... ]` (optional `?status=RUNNING\|JUDGING\|PASSED\|FAILED` — filtered client-side from `getAllRuns`) | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/runs/{id}` | — | `200 EvalRun` / `404` | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` (one event per run change) | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/runs`

- Missing `passThreshold` → `0.75` (from `multi-agent-eval.run.default-pass-threshold`).
- Missing `requestedBy` → `"anonymous"`.
- `passThreshold` must be in `(0.0, 1.0]`; otherwise `400`.
- Duplicate-detection window: 15 s on `(suiteName, requestedBy)`; the second submission returns the first `runId` (200) instead of starting a new workflow.

## JSON shapes

### EvalRun

```json
{
  "runId": "r-3a9f1…",
  "suiteName": "reasoning-baseline",
  "passThreshold": 0.75,
  "maxRetries": 2,
  "status": "PASSED",
  "scenarioResults": [
    {
      "scenarioId": "S-1",
      "agentResponse": "The answer is 42 because …",
      "thresholdMet": true,
      "completedAt": "2026-06-28T10:02:14Z"
    },
    {
      "scenarioId": "S-3",
      "agentResponse": "At 60 mph over 790 miles …",
      "thresholdMet": true,
      "completedAt": "2026-06-28T10:02:19Z"
    }
  ],
  "judgments": [
    {
      "verdict": "PASS",
      "notes": {
        "bullets": [],
        "overallRationale": "All scenarios responded; 5 of 6 thresholdMet; no safety failures; no contradictions."
      },
      "score": 0.92,
      "evaluatedAt": "2026-06-28T10:02:31Z"
    }
  ],
  "finalScore": 0.92,
  "failureReason": null,
  "ciGateBlocked": false,
  "createdAt": "2026-06-28T10:02:00Z",
  "finishedAt": "2026-06-28T10:02:32Z"
}
```

### CIGateBlocked event form

```json
{
  "runId": "r-3a9f1…",
  "finalScore": 0.62,
  "passThreshold": 0.75,
  "blockedAt": "2026-06-28T10:05:11Z"
}
```

### SSE event format

```
event: run-update
data: { "runId": "r-3a9f1…", "status": "JUDGING", "scenarioResults": [...], "judgments": [...], ... }
```

One event per state transition. Clients reconcile by `runId`. The full `EvalRun` JSON is included so a fresh client can render the row without a separate fetch.

### CI gate integration

External CI pipelines have two integration points:

1. **SSE subscription** — `GET /api/runs/sse` emits `run-update` events; filter on `ciGateBlocked: true`.
2. **Polling** — `GET /api/runs/{id}` returns the full run; check `ciGateBlocked` field after the status reaches `PASSED` or `FAILED`.

A pipeline that blocks promotion should gate on `ciGateBlocked === true`, not on `status === "FAILED"` — a run can reach `PASSED` with a `ciGateBlocked` flag if the aggregate score narrowly satisfied the judge rubric but fell below the deployer's configured threshold.
