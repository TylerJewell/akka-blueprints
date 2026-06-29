# API contract — mle-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `{ "datasetRef": String, "objective": String, "requestedBy"?: String }` | `202 { "runId": String }` | `PipelineEndpoint` → `RunQueue` |
| `GET` | `/api/runs` | — | `200 [ RunRow... ]` | `PipelineEndpoint` ← `PipelineRunView` |
| `GET` | `/api/runs/{id}` | — | `200 PipelineRun` / `404` | `PipelineEndpoint` ← `PipelineRunEntity` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` (one event per run change) | `PipelineEndpoint` ← `PipelineRunView` |
| `GET` | `/api/alerts` | — | `200 [ DriftAlert... ]` (newest first) | `PipelineEndpoint` ← `PipelineRunView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `PipelineEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `PipelineEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `PipelineEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### PipelineRun (full form, returned by `GET /api/runs/{id}`)

```json
{
  "runId": "r-3a9…",
  "datasetRef": "credit-risk-schema.json",
  "objective": "binary classification",
  "status": "EXECUTING",
  "ledger": {
    "facts": ["dataset is credit-risk schema", "objective is binary classification"],
    "gaps": ["best encoding for categorical columns"],
    "stages": [
      "Profile credit-risk-schema.json",
      "Engineer features",
      "Train gradient-boosted classifier",
      "Evaluate model",
      "Promote if gate passes"
    ],
    "activeDispatch": {
      "specialist": "MODEL_TRAINER",
      "stageName": "train-classifier",
      "instruction": "Train with 100 estimators, max depth 5",
      "rationale": "Feature plan complete; ready to train."
    }
  },
  "evalLedger": {
    "entries": [
      {
        "attempt": 1,
        "specialist": "DATA_PROFILER",
        "stageName": "profile-dataset",
        "verdict": "OK",
        "metrics": null,
        "gateOutcome": "NOT_APPLICABLE",
        "blocker": null,
        "recordedAt": "2026-06-28T10:01:12Z"
      },
      {
        "attempt": 1,
        "specialist": "FEATURE_ENGINEER",
        "stageName": "engineer-features",
        "verdict": "OK",
        "metrics": null,
        "gateOutcome": "NOT_APPLICABLE",
        "blocker": null,
        "recordedAt": "2026-06-28T10:01:29Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "driftAlerts": [],
  "createdAt": "2026-06-28T10:01:05Z",
  "finishedAt": null
}
```

### RunRow (list form, returned by `GET /api/runs`)

`RunRow` mirrors `PipelineRun`, but `evalLedger.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full run by id on row expand.

### DriftAlert (returned in `GET /api/alerts` and embedded in `PipelineRun.driftAlerts`)

```json
{
  "runId": "r-3a9…",
  "metricName": "auc",
  "baseline": 0.91,
  "observed": 0.79,
  "delta": 0.12,
  "detectedAt": "2026-06-28T10:15:00Z"
}
```

### SSE event format

```
event: run-update
data: { "runId": "r-3a9…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `runId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating gate-failure spike", "haltedAt": "2026-06-28T10:22:07Z" }
```

And `event: drift-alert` whenever `DriftAlertRaised` fires:

```
event: drift-alert
data: { "runId": "r-3a9…", "metricName": "auc", "baseline": 0.91, "observed": 0.79, "delta": 0.12, "detectedAt": "2026-06-28T10:15:00Z" }
```
