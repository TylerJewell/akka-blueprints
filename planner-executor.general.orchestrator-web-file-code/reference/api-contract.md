# API contract — orchestrator-web-file-code

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "taskId": String }` | `TaskEndpoint` → `RequestQueue` |
| `GET` | `/api/tasks` | — | `200 [ Task... ]` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `TaskEndpoint` ← `TaskEntity` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `TaskEndpoint` ← `TaskView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `TaskEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `TaskEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `TaskEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Task (full form, returned by `GET /api/tasks/{id}`)

```json
{
  "taskId": "t-7f2…",
  "prompt": "Find the latest Akka release notes and summarise new agent features.",
  "status": "EXECUTING",
  "ledger": {
    "facts": ["the user wants a summary of new agent features"],
    "missing": ["which release is current"],
    "plan": [
      "Web-search for the latest Akka release",
      "File-read the agent-features section of the release notes",
      "Compose a 100-word summary"
    ],
    "currentDispatch": {
      "specialist": "WEB",
      "subtask": "latest Akka release version",
      "rationale": "Need to anchor the search to the current version."
    }
  },
  "progress": {
    "entries": [
      {
        "attempt": 1,
        "specialist": "WEB",
        "subtask": "latest Akka release version",
        "verdict": "OK",
        "scrubbedResult": "Akka 3.6.0 (akka.io, 'Release notes')...",
        "blocker": null,
        "recordedAt": "2026-06-28T07:33:01Z"
      },
      {
        "attempt": 1,
        "specialist": "FILE",
        "subtask": "agent-features section of release notes",
        "verdict": "OK",
        "scrubbedResult": "sample-data/files/release-notes.md\n...AutonomousAgent now supports...",
        "blocker": null,
        "recordedAt": "2026-06-28T07:33:09Z"
      }
    ]
  },
  "answer": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T07:32:55Z",
  "finishedAt": null
}
```

### Task (list form, returned by `GET /api/tasks`)

The list form is `TaskRow` — same fields as `Task`, but `progress.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full task by id on row expand.

### SSE event format

```
event: task-update
data: { "taskId": "t-7f2…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `taskId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating spike in BLOCKED entries", "haltedAt": "2026-06-28T07:34:21Z" }
```
