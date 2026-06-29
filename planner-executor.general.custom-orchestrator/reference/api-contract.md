# API contract — custom-orchestration-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "taskId": String }` | `TaskEndpoint` → `RequestQueue` |
| `GET` | `/api/tasks` | — | `200 [ TaskRow... ]` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `TaskEndpoint` ← `TaskEntity` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `TaskEndpoint` ← `TaskView` |
| `POST` | `/api/strategies` | `OrchestrationStrategy` | `201 { "name": String }` | `TaskEndpoint` → `StrategyRegistry` |
| `POST` | `/api/strategies/activate/{name}` | — | `200 { "active": String }` / `404` | `TaskEndpoint` → `StrategyRegistry` |
| `GET` | `/api/strategies` | — | `200 [ StrategyEntry... ]` | `TaskEndpoint` ← `StrategyRegistry` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `TaskEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `TaskEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `TaskEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (`index.html`) | `AppEndpoint` |

## JSON shapes

### Task (full form, returned by `GET /api/tasks/{id}`)

```json
{
  "taskId": "t-9a3…",
  "prompt": "Research the latest Akka SDK release and produce a changelog summary.",
  "status": "EXECUTING",
  "activeStrategy": {
    "name": "default",
    "description": "General-purpose routing; all tools available.",
    "maxRouteIterations": 10,
    "maxRevisits": 2,
    "allowedTools": ["search", "file", "code", "command"]
  },
  "context": {
    "goal": "Research the latest Akka SDK release and produce a changelog summary.",
    "strategy": { "name": "default", "..." : "..." },
    "traceEntries": [
      {
        "sequence": 1,
        "kind": "TOOL_CALL",
        "toolCall": { "toolName": "search", "args": "latest Akka SDK release", "rationale": "Find the current version." },
        "scrubbedResult": "Akka 3.6.0 released (akka.io, 'Release notes')...",
        "verdict": "OK",
        "blocker": null,
        "recordedAt": "2026-06-28T09:10:05Z"
      }
    ]
  },
  "answer": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:09:58Z",
  "finishedAt": null
}
```

### TaskRow (list form, returned by `GET /api/tasks`)

`TaskRow` mirrors `Task` but `context.traceEntries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. Each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full task by id on row expand.

### StrategyEntry (returned by `GET /api/strategies`)

```json
{
  "name": "default",
  "description": "General-purpose routing; all tools available.",
  "maxRouteIterations": 10,
  "maxRevisits": 2,
  "allowedTools": ["search", "file", "code", "command"],
  "active": true
}
```

### SSE event formats

```
event: task-update
data: { "taskId": "t-9a3…", "status": "EXECUTING", ... }
```

One event per state transition or trace append. Clients reconcile by `taskId`. The SSE channel also emits `event: control-update` when the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "testing new strategy", "haltedAt": "2026-06-28T09:12:00Z" }
```
