# API contract — akka-basic

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `PromptEndpoint` → `PromptEntity` |
| `GET` | `/api/runs` | — | `200 [ Run... ]` (newest-first) | `PromptEndpoint` ← `PromptView` |
| `GET` | `/api/runs/{id}` | — | `200 Run` / `404` | `PromptEndpoint` ← `PromptView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `PromptEndpoint` ← `PromptView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `PromptEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `PromptEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `PromptEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "promptText": "Summarise the following quarterly report in two sentences: Revenue grew 12% year-over-year...",
  "categoryHint": "SUMMARISE",
  "submittedBy": "analyst-7"
}
```

`categoryHint` is one of `SUMMARISE`, `EXTRACT`, `CLASSIFY`, `AUTO`.

### Run (response body)

```json
{
  "runId": "run-3ac...",
  "request": {
    "runId": "run-3ac...",
    "promptText": "Summarise the following quarterly report in two sentences: Revenue grew 12% year-over-year...",
    "categoryHint": "SUMMARISE",
    "submittedBy": "analyst-7",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "result": {
    "outputText": "The report documents a 12% year-over-year revenue increase driven by enterprise subscriptions.",
    "confidenceScore": 0.88,
    "category": "SUMMARISE",
    "producedAt": "2026-06-28T14:00:18Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: run-update
data: { "runId": "run-3ac...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PROCESSING`, `COMPLETED`, `FAILED`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity should wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
