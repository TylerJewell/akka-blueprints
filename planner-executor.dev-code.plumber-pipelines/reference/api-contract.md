# API contract — plumber-data-engineering-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/pipelines` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "pipelineId": String }` | `PipelineEndpoint` → `RequestQueue` |
| `GET` | `/api/pipelines` | — | `200 [ PipelineRow... ]` | `PipelineEndpoint` ← `PipelineView` |
| `GET` | `/api/pipelines/{id}` | — | `200 Pipeline` / `404` | `PipelineEndpoint` ← `PipelineEntity` |
| `GET` | `/api/pipelines/sse` | — | `text/event-stream` (one event per pipeline change) | `PipelineEndpoint` ← `PipelineView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `PipelineEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `PipelineEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `PipelineEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Pipeline (full form, returned by `GET /api/pipelines/{id}`)

```json
{
  "pipelineId": "p-4a1…",
  "prompt": "Design a Spark pipeline that reads Parquet from s3://raw-events, filters nulls, and writes Avro to s3://clean-events.",
  "status": "EXECUTING",
  "ledger": {
    "sources": ["s3://raw-events [parquet, schema:raw-clickstream-v2]"],
    "transforms": ["filter rows where user_id IS NULL", "cast event_ts to timestamp"],
    "sinks": ["s3://clean-events [avro, schema:clean-clickstream-v1]"],
    "validationPlan": [
      "validate source schema raw-clickstream-v2",
      "validate sink schema clean-clickstream-v1"
    ],
    "currentStage": {
      "engine": "SPARK",
      "stageKind": "SOURCE",
      "spec": "Generate a Spark DataFrameReader config for s3://raw-events in Parquet format.",
      "rationale": "First stage: read the source dataset."
    }
  },
  "progress": {
    "entries": [
      {
        "attempt": 1,
        "engine": "SCHEMA",
        "stageKind": "VALIDATE",
        "verdict": "OK",
        "artifact": "{\"checkedSchemas\":[\"raw-clickstream-v2\"],\"compatible\":true,\"findings\":[]}",
        "blocker": null,
        "recordedAt": "2026-06-28T08:12:03Z"
      }
    ]
  },
  "definition": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T08:11:55Z",
  "finishedAt": null
}
```

### Pipeline (list form, returned by `GET /api/pipelines`)

The list form is `PipelineRow` — same fields as `Pipeline`, but `progress.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full pipeline by id on row expand.

### SSE event format

```
event: pipeline-update
data: { "pipelineId": "p-4a1…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `pipelineId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "reviewing test-gate failures", "haltedAt": "2026-06-28T08:14:10Z" }
```
