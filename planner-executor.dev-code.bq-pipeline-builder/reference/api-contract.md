# API contract — bq-pipeline-builder

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/pipelines` | `{ "description": String, "requestedBy"?: String }` | `202 { "pipelineId": String }` | `PipelineEndpoint` → `PipelineQueue` |
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
  "pipelineId": "p-3a9…",
  "description": "Build a pipeline that loads raw GA4 events into a clean sessions table.",
  "status": "BUILDING",
  "ledger": {
    "datasetFacts": ["source is raw.ga4_events", "target is mart.sessions"],
    "schemaGaps": ["session window definition not confirmed"],
    "buildPlan": [
      "Analyze raw.ga4_events schema",
      "Compose sessions aggregation SQL",
      "Model sessions as a Dataform SQLX definition",
      "Validate the SQLX definition"
    ],
    "currentDispatch": {
      "executor": "SQL_COMPOSER",
      "step": "Draft SELECT … GROUP BY session_id aggregation on ga4_events",
      "rationale": "Schema known; ready to compose the core aggregation."
    }
  },
  "steps": {
    "entries": [
      {
        "attempt": 1,
        "executor": "SCHEMA_ANALYST",
        "step": "Analyze raw.ga4_events schema",
        "verdict": "OK",
        "output": "Table: raw.ga4_events\nColumns:\n  event_id (STRING, REQUIRED) — unique event identifier\n  session_id (STRING, NULLABLE) — session grouping key\n  event_timestamp (TIMESTAMP, REQUIRED) — when the event fired\n  user_pseudo_id (STRING, NULLABLE) — anonymous user identifier\n  event_name (STRING, REQUIRED) — GA4 event type label",
        "blocker": null,
        "recordedAt": "2026-06-28T09:11:03Z"
      }
    ]
  },
  "manifest": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:10:55Z",
  "finishedAt": null
}
```

### Pipeline (list form, returned by `GET /api/pipelines`)

The list form is `PipelineRow` — same fields as `Pipeline`, but `steps.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full pipeline by id on row expand.

### SSE event format

```
event: pipeline-update
data: { "pipelineId": "p-3a9…", "status": "BUILDING", ... }
```

One event per state transition. Clients reconcile by `pipelineId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating repeated CI gate failures", "haltedAt": "2026-06-28T09:14:07Z" }
```
