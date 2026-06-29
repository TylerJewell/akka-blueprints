# API contract — rewoo-fixed-plan

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `{ "query": String, "requestedBy"?: String }` | `202 { "queryId": String }` | `QueryEndpoint` → `RequestQueue` |
| `GET` | `/api/queries` | — | `200 [ QueryRow... ]` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryEntity` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` (one event per query change) | `QueryEndpoint` ← `QueryView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `QueryEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `QueryEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `QueryEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Query (full form, returned by `GET /api/queries/{id}`)

```json
{
  "queryId": "q-4a1…",
  "query": "What is the current Akka version and how does it compare to the release six months ago?",
  "status": "EXECUTING",
  "plan": {
    "steps": [
      {
        "stepIndex": 0,
        "tool": "WEB_SEARCH",
        "inputExpression": "latest Akka release version from akka.io/releases",
        "result": "Akka 3.6.0 (released 2026-05-12)"
      },
      {
        "stepIndex": 1,
        "tool": "FILE_READ",
        "inputExpression": "Read sample-data/files/release-archive.md for the release dated six months ago",
        "result": null
      },
      {
        "stepIndex": 2,
        "tool": "CALCULATE",
        "inputExpression": "Compare #E0 and #E1: identify new features and breaking changes",
        "result": null
      }
    ]
  },
  "stepRecords": [
    {
      "stepIndex": 0,
      "tool": "WEB_SEARCH",
      "resolvedInput": "latest Akka release version from akka.io/releases",
      "verdict": "COMPLETED",
      "scrubbedResult": "Akka 3.6.0 (released 2026-05-12)",
      "blockReason": null,
      "recordedAt": "2026-06-28T09:11:03Z"
    }
  ],
  "answer": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:10:58Z",
  "finishedAt": null
}
```

### QueryRow (list form, returned by `GET /api/queries`)

`QueryRow` mirrors `Query` but `stepRecords` is truncated to the last 3 entries plus `truncatedFromTotal: int`. Each entry's `scrubbedResult` is capped at 240 characters. The `plan.steps` list is included but each `result` field is truncated to 120 characters. The UI fetches the full query by id on row expand.

### SSE event format

```
event: query-update
data: { "queryId": "q-4a1…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `queryId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating unexpected FILE_READ path", "haltedAt": "2026-06-28T09:14:11Z" }
```
