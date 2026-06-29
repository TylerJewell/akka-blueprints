# API contract — agent-eval-harness

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `{ "suiteName": String, "passingThreshold"?: Double }` | `202 { "runId": String }` | `EvalEndpoint` → `SuiteRegistry` |
| `GET` | `/api/runs` | — | `200 [ EvalRun... ]` (optional `?status=PENDING\|RUNNING\|PASSED\|FAILED` — filtered client-side from `getAllRuns`) | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/runs/{id}` | — | `200 EvalRun` / `404` | `EvalEndpoint` ← `EvalRunsView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` (one event per run change) | `EvalEndpoint` ← `EvalRunsView` |
| `POST` | `/api/suites` | `{ "suiteName": String, "cases": TestCase[], "rubric": String, "defaultPassingThreshold"?: Double }` | `201 { "suiteName": String }` | `EvalEndpoint` → `SuiteRegistry` |
| `GET` | `/api/suites/{name}` | — | `200 Suite` / `404` | `EvalEndpoint` ← `SuiteRegistry` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/runs`

- Missing `passingThreshold` → reads `suite.defaultPassingThreshold` from the registered suite; falls back to `0.8` (from `agent-eval-harness.eval.default-passing-threshold`).
- `passingThreshold` must be in `(0.0, 1.0]`; otherwise `400`.
- Unknown `suiteName` → `404`.

## Defaults on `POST /api/suites`

- Missing `defaultPassingThreshold` → `0.8`.
- `cases` must be non-empty; otherwise `400`.
- Each `TestCase.caseId` must be unique within the suite; otherwise `400`.
- Registering an existing `suiteName` replaces the prior definition.

## JSON shapes

### EvalRun

```json
{
  "runId": "r-9b3c1…",
  "suiteName": "default",
  "totalCases": 5,
  "passingThreshold": 0.8,
  "status": "PASSED",
  "results": [
    {
      "caseId": "q-001",
      "response": {
        "caseId": "q-001",
        "rawOutput": "Paris",
        "respondedAt": "2026-06-28T09:00:01Z"
      },
      "judgment": {
        "verdict": "PASS",
        "notes": {
          "bullets": [],
          "overallRationale": "Output matches expected answer exactly."
        },
        "confidenceScore": 0.95,
        "judgedAt": "2026-06-28T09:00:04Z"
      }
    },
    {
      "caseId": "q-002",
      "response": {
        "caseId": "q-002",
        "rawOutput": "Water evaporates, condenses, and precipitates.",
        "respondedAt": "2026-06-28T09:00:12Z"
      },
      "judgment": {
        "verdict": "FAIL",
        "notes": {
          "bullets": [
            "Output omits 'condensation' as a distinct stage — rubric requires explicit mention.",
            "Sentence is otherwise grammatically correct."
          ],
          "overallRationale": "Completeness fails on one required rubric term."
        },
        "confidenceScore": 0.82,
        "judgedAt": "2026-06-28T09:00:15Z"
      }
    }
  ],
  "passRate": 0.8,
  "passCount": 4,
  "failCount": 1,
  "failureReason": null,
  "startedAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:01:02Z"
}
```

### TestCase

```json
{
  "caseId": "q-001",
  "input": "What is the capital of France?",
  "expectedAnswer": "Paris",
  "rubricHint": "single-word city name"
}
```

### SSE event format

```
event: run-update
data: { "runId": "r-9b3c1…", "status": "RUNNING", "results": [...], ... }
```

One event per state transition and per `CaseEvaluated` write. Clients reconcile by `runId`. The full `EvalRun` JSON is included so a fresh client can render the row without a separate fetch.
