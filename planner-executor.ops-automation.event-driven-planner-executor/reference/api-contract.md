# API contract — event-driven-planner-executor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "jobId": String }` | `JobEndpoint` → `RequestQueue` |
| `GET` | `/api/jobs` | — | `200 [ Job... ]` | `JobEndpoint` ← `JobView` |
| `GET` | `/api/jobs/{id}` | — | `200 Job` / `404` | `JobEndpoint` ← `JobEntity` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `JobEndpoint` ← `JobView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `JobEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `JobEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `JobEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Job (full form, returned by `GET /api/jobs/{id}`)

```json
{
  "jobId": "j-3a1…",
  "prompt": "Fetch the latest deployment manifest from the registry, publish a rollout event, query the active instance count, and summarise the state.",
  "status": "EXECUTING",
  "ledger": {
    "facts": ["the operator wants a summary of the current deployment state"],
    "missing": ["latest manifest version", "rollout event topic"],
    "plan": [
      "HTTP-call the manifest registry endpoint",
      "QUEUE-publish a rollout event to the deployments topic",
      "DB-query the active instance count",
      "Compose a status report"
    ],
    "currentDispatch": {
      "executor": "HTTP",
      "step": "fetch latest manifest from registry.example.io/api/v1/manifests/latest",
      "rationale": "Need the manifest version to anchor the rollout."
    }
  },
  "steps": {
    "entries": [
      {
        "attempt": 1,
        "executor": "HTTP",
        "step": "fetch latest manifest from registry.example.io/api/v1/manifests/latest",
        "verdict": "OK",
        "scrubbedResult": "200 OK — manifest v1.14.3, image: akka/compliance-surface:1.14.3, replicas: 3",
        "blocker": null,
        "recordedAt": "2026-06-28T10:05:12Z"
      },
      {
        "attempt": 1,
        "executor": "QUEUE",
        "step": "publish rollout event to deployments topic",
        "verdict": "OK",
        "scrubbedResult": "Published to deployments at partition 0, offset 4821, timestamp 2026-06-28T10:05:18Z",
        "blocker": null,
        "recordedAt": "2026-06-28T10:05:19Z"
      }
    ]
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:05:07Z",
  "finishedAt": null
}
```

### Job (list form, returned by `GET /api/jobs`)

The list form is `JobRow` — same fields as `Job`, but `steps.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full job by id on row expand.

### SSE event format

```
event: job-update
data: { "jobId": "j-3a1…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `jobId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "elevated retry count on DB steps", "haltedAt": "2026-06-28T10:07:41Z" }
```
