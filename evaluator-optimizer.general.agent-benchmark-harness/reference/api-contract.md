# API contract — agent-benchmark-harness

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `{ "triggeredBy"?: String, "taskFilter"?: [String] }` | `202 { "runId": String }` | `BenchmarkEndpoint` → `BenchmarkRunWorkflow` |
| `GET` | `/api/runs` | — | `200 [ BenchmarkRun... ]` (optional `?status=PENDING\|RUNNING\|PASSED\|FAILED` — filtered client-side from `getAllRuns`) | `BenchmarkEndpoint` ← `RunsView` |
| `GET` | `/api/runs/{id}` | — | `200 BenchmarkRun` / `404` | `BenchmarkEndpoint` ← `RunsView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` (one event per run state change) | `BenchmarkEndpoint` ← `RunsView` |
| `GET` | `/api/tasks` | — | `200 [ EvalTask... ]` | `BenchmarkEndpoint` ← `TaskRegistry` |
| `GET` | `/api/ci-gate` | — | `200 { "gated": boolean, "runId": String\|null, "passRate": double\|null, "threshold": double, "reason"?: String }` | `BenchmarkEndpoint` ← `RunsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `BenchmarkEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `BenchmarkEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `BenchmarkEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/runs`

- Missing `triggeredBy` → `"anonymous"`.
- Missing `taskFilter` → all registered tasks are included.
- `taskFilter` is a list of category strings (`"reasoning"`, `"factual"`, `"instruction-following"`); only tasks matching at least one listed category are included.
- If the task set after filtering is empty, the request returns `400 { "error": "No tasks match the specified filter" }`.

## `GET /api/ci-gate` semantics

- `gated: false` — most recent completed run has `passRate >= threshold`. Safe to proceed.
- `gated: true` — most recent completed run has `passRate < threshold`, or no completed run exists. Do not proceed.
- `reason` field is present only when `gated: true`. Values: `"pass rate N% below threshold M%"`, `"no completed run"`.

## JSON shapes

### BenchmarkRun

```json
{
  "runId": "run-a4f91…",
  "triggeredBy": "anonymous",
  "status": "PASSED",
  "taskAttempts": [
    {
      "taskId": "t-001",
      "response": {
        "taskId": "t-001",
        "rawOutput": "The capital of France is Paris.",
        "respondedAt": "2026-06-28T10:15:33Z"
      },
      "scored": {
        "taskId": "t-001",
        "verdict": "PASS",
        "score": 100,
        "rationale": "Response states Paris — matches reference answer exactly.",
        "scoredAt": "2026-06-28T10:15:41Z"
      }
    },
    {
      "taskId": "t-002",
      "response": {
        "taskId": "t-002",
        "rawOutput": "The halting problem is undecidable because no general algorithm can determine whether an arbitrary program halts.",
        "respondedAt": "2026-06-28T10:15:52Z"
      },
      "scored": {
        "taskId": "t-002",
        "verdict": "PASS",
        "score": 92,
        "rationale": "Conclusion and key reasoning step are correct; minor elaboration beyond reference.",
        "scoredAt": "2026-06-28T10:16:01Z"
      }
    }
  ],
  "aggregate": {
    "totalTasks": 2,
    "passCount": 2,
    "failCount": 0,
    "passRate": 1.0,
    "durationMs": 28340
  },
  "failureReason": null,
  "startedAt": "2026-06-28T10:15:30Z",
  "finishedAt": "2026-06-28T10:16:02Z"
}
```

### CI gate response examples

```json
{ "gated": false, "runId": "run-a4f91…", "passRate": 1.0,  "threshold": 0.80 }
{ "gated": true,  "runId": "run-b2e77…", "passRate": 0.60,  "threshold": 0.80, "reason": "pass rate 60% below threshold 80%" }
{ "gated": true,  "runId": null,          "passRate": null,  "threshold": 0.80, "reason": "no completed run" }
```

### SSE event format

```
event: run-update
data: { "runId": "run-a4f91…", "status": "RUNNING", "taskAttempts": [...], "aggregate": null, ... }
```

One event per state transition. Clients reconcile by `runId`. The full `BenchmarkRun` JSON is included so a fresh client can render the row without a separate fetch.

### EvalTask

```json
{
  "taskId": "t-001",
  "prompt": "What is the capital of France?",
  "referenceAnswer": "Paris",
  "category": "factual"
}
```
