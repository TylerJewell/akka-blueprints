# API contract — structured-mcp-tool

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { queryId }` | `QueryEndpoint` → `QueryEntity` |
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
  "location": "Tokyo",
  "unit": "CELSIUS",
  "submittedBy": "demo-user-1"
}
```

### Query (response body — COMPLETED state)

```json
{
  "queryId": "q-7fa...",
  "request": {
    "queryId": "q-7fa...",
    "location": "Tokyo",
    "unit": "CELSIUS",
    "submittedBy": "demo-user-1",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "summary": {
    "location": "Tokyo",
    "temperatureCelsius": 28.1,
    "temperatureFahrenheit": null,
    "humidityPercent": 65,
    "windSpeedKph": 11.5,
    "condition": "PARTLY_CLOUDY",
    "narrative": "Tokyo is partly cloudy at 28.1 °C with light winds at 11.5 km/h.",
    "returnedAt": "2026-06-28T10:00:22Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:22Z"
}
```

### Query (response body — SUBMITTED state, before agent returns)

```json
{
  "queryId": "q-7fa...",
  "request": {
    "queryId": "q-7fa...",
    "location": "Tokyo",
    "unit": "CELSIUS",
    "submittedBy": "demo-user-1",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "summary": null,
  "status": "SUBMITTED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: query-update
data: { "queryId": "q-7fa...", "status": "COMPLETED", "summary": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `COMPLETED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `submittedBy` from the authenticated principal rather than the request body.
