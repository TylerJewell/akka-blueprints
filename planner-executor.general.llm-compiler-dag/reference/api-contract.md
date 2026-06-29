# API contract — llm-compiler-dag

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "query": String, "requestedBy"?: String }` | `202 { "jobId": String }` | `JobEndpoint` → `RequestQueue` |
| `GET` | `/api/jobs` | — | `200 [ JobRow... ]` | `JobEndpoint` ← `JobView` |
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
  "jobId": "j-3a9…",
  "query": "What is (42 * 17) + the minor version of the current Akka release?",
  "status": "RUNNING",
  "plan": {
    "description": "Compute 42 × 17, look up the current Akka minor version, then add them.",
    "toolCalls": [
      { "callId": "c1", "tool": "CALCULATOR", "arguments": { "expression": "42 * 17" }, "dependsOn": [] },
      { "callId": "c2", "tool": "SEARCH",     "arguments": { "query": "current Akka release version" }, "dependsOn": [] },
      { "callId": "c3", "tool": "CALCULATOR", "arguments": { "expression": "714 + 6" }, "dependsOn": ["c1", "c2"] }
    ]
  },
  "results": {
    "results": [
      {
        "callId": "c1",
        "tool": "CALCULATOR",
        "ok": true,
        "output": "714",
        "errorReason": null
      },
      {
        "callId": "c2",
        "tool": "SEARCH",
        "ok": true,
        "output": "Current Akka release is 3.6.0 (minor = 6). (akka.io, 'Akka 3.6.0 release notes')",
        "errorReason": null
      }
    ]
  },
  "answer": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:11:02Z",
  "finishedAt": null
}
```

### JobRow (list form, returned by `GET /api/jobs`)

`JobRow` mirrors `Job` but `results.results` is truncated to the last 3 `ToolResult` entries plus `truncatedFromTotal: <int>`. Each `output` is capped at 240 characters. The UI fetches the full job by id on row expand via `GET /api/jobs/{id}`.

### SSE event format

```
event: job-update
data: { "jobId": "j-3a9…", "status": "RUNNING", "frontierSize": 2, ... }
```

One event per state transition or per batch of resolved tool calls. Clients reconcile by `jobId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "unusual search pattern observed", "haltedAt": "2026-06-28T09:12:44Z" }
```
