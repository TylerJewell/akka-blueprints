# API contract — akka-stategraph-bridge

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `{ "graphJson": String, "requestedBy"?: String }` | `202 { "runId": String }` | `GraphEndpoint` → `RunQueueEntity` |
| `GET` | `/api/runs` | — | `200 [ GraphRunRow... ]` | `GraphEndpoint` ← `GraphRunView` |
| `GET` | `/api/runs/{id}` | — | `200 GraphRun` / `404` | `GraphEndpoint` ← `GraphRunEntity` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` (one event per run change) | `GraphEndpoint` ← `GraphRunView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `GraphEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `GraphEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `GraphEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### GraphRun (full form, returned by `GET /api/runs/{id}`)

```json
{
  "runId": "r-9a3…",
  "graphJson": "{\"nodes\":[...],\"edges\":[...]}",
  "status": "EXECUTING",
  "plan": {
    "nodes": [
      { "nodeId": "fetch-docs", "description": "Fetch latest release notes", "toolAnnotation": "doc.akka.io/fetch", "maxVisits": 1 },
      { "nodeId": "summarise", "description": "Summarise findings", "toolAnnotation": null, "maxVisits": 1 },
      { "nodeId": "output", "description": "Produce final output", "toolAnnotation": null, "maxVisits": 1 }
    ],
    "edges": [
      { "fromNodeId": "fetch-docs", "toNodeId": "summarise", "predicate": null, "isBackEdge": false },
      { "fromNodeId": "summarise", "toNodeId": "output", "predicate": null, "isBackEdge": false }
    ],
    "entryNodeId": "fetch-docs",
    "terminalNodeIds": ["output"],
    "cycleAnnotations": []
  },
  "currentState": {
    "fields": {
      "release_version": "3.6.0",
      "release_url": "https://doc.akka.io/release-notes/3.6.0.html"
    }
  },
  "trace": [
    {
      "stepIndex": 1,
      "nodeId": "fetch-docs",
      "verdict": "OK",
      "scrubbedContent": "Fetched release notes for 3.6.0 from doc.akka.io...",
      "blocker": null,
      "recordedAt": "2026-06-28T10:14:02Z"
    }
  ],
  "result": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:13:55Z",
  "finishedAt": null
}
```

### GraphRunRow (list form, returned by `GET /api/runs`)

`GraphRunRow` mirrors `GraphRun` but `trace` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. Each `scrubbedContent` is capped at 240 characters. The UI fetches the full run by id on row expand.

### SSE event format

```
event: run-update
data: { "runId": "r-9a3…", "status": "EXECUTING", "currentState": { "fields": { ... } } }
```

One event per state transition. Clients reconcile by `runId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating unexpected tool call", "haltedAt": "2026-06-28T10:15:30Z" }
```
