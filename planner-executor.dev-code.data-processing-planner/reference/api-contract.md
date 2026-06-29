# API contract — data-processing-planner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/pipelines` | `{ "description": String, "targetDataset": String, "requestedBy"?: String }` | `202 { "runId": String }` | `PipelineEndpoint` → `PipelineQueue` |
| `GET` | `/api/pipelines` | — | `200 [ PipelineRunRow... ]` | `PipelineEndpoint` ← `PipelineView` |
| `GET` | `/api/pipelines/{id}` | — | `200 PipelineRun` / `404` | `PipelineEndpoint` ← `PipelineEntity` |
| `GET` | `/api/pipelines/sse` | — | `text/event-stream` (one event per run state change) | `PipelineEndpoint` ← `PipelineView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `PipelineEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `PipelineEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `PipelineEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### PipelineRun (full form, returned by `GET /api/pipelines/{id}`)

```json
{
  "runId": "pr-4a1…",
  "description": "Crawl raw-events, EMR de-duplicate, write to processed/",
  "targetDataset": "raw-events",
  "status": "RUNNING",
  "ledger": {
    "knownInputs": ["raw-events S3 prefix s3://dev-lake/raw-events/"],
    "missingParams": ["de-duplication key column"],
    "jobPlan": [
      "GLUE_CRAWLER: catalog raw-events prefix to discover schema",
      "EMR_STEP: de-duplicate on discovered key, write to s3://dev-output/processed/",
      "S3_COPY: copy manifest to s3://dev-audit/manifests/"
    ],
    "currentDispatch": {
      "engine": "EMR_STEP",
      "stepSpec": "de-duplicate on event_id, write to s3://dev-output/processed/",
      "estimatedCostUsd": 2.50,
      "rationale": "Schema known; EMR is cheapest de-dup engine for this dataset size."
    }
  },
  "runLedger": {
    "entries": [
      {
        "attempt": 1,
        "engine": "GLUE_CRAWLER",
        "stepSpec": "catalog raw-events prefix to discover schema",
        "verdict": "OK",
        "scrubbedOutput": "Discovered table raw_events_catalog: 14 columns, partition key=event_date, ~4 M rows",
        "costConsumed": 0.30,
        "blocker": null,
        "recordedAt": "2026-06-28T10:14:02Z"
      }
    ]
  },
  "output": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:13:55Z",
  "finishedAt": null
}
```

### PipelineRunRow (list form, returned by `GET /api/pipelines`)

`PipelineRunRow` mirrors `PipelineRun` but `runLedger.entries` is truncated to the last 3 entries plus `truncatedFromTotal: int`. Each entry's `scrubbedOutput` is capped at 240 characters. The UI fetches the full run by id on row expand.

### SSE event format

```
event: run-update
data: { "runId": "pr-4a1…", "status": "RUNNING", ... }
```

One event per state transition. Clients reconcile by `runId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating cost spike", "haltedAt": "2026-06-28T10:17:34Z" }
```
