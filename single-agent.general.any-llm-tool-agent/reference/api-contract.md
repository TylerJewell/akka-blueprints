# API contract — any-llm-tool-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` + `QueryWorkflow` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "queryText": "What is the weather in Tokyo right now?",
  "requestedBackend": "mock",
  "submittedBy": "demo-user"
}
```

### Query (response body — RESULT_RECORDED state)

```json
{
  "queryId": "q-7ac...",
  "query": {
    "queryId": "q-7ac...",
    "queryText": "What is the weather in Tokyo right now?",
    "requestedBackend": "mock",
    "submittedBy": "demo-user",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "report": {
    "location": "Tokyo",
    "data": {
      "location": "Tokyo",
      "condition": "SUNNY",
      "temperatureCelsius": 28.0,
      "humidityPercent": 60,
      "windSpeedKmh": 10.0
    },
    "narrative": "Tokyo is currently sunny with a temperature of 28°C, 60% humidity, and a light wind of 10 km/h.",
    "reportedAt": "2026-06-28T14:00:15Z"
  },
  "status": "RESULT_RECORDED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:15Z"
}
```

### Query (response body — SUBMITTED state, partial)

```json
{
  "queryId": "q-7ac...",
  "query": {
    "queryId": "q-7ac...",
    "queryText": "What is the weather in Tokyo right now?",
    "requestedBackend": "mock",
    "submittedBy": "demo-user",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "report": null,
  "status": "SUBMITTED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7ac...", "status": "RESULT_RECORDED", "report": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `INVOKING`, `RESULT_RECORDED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
