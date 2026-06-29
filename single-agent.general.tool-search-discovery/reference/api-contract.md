# API contract — tool-search-discovery

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `SubmitTaskRequest` | `201 { taskId }` | `TaskEndpoint` → `TaskEntity` |
| `GET` | `/api/tasks` | — | `200 [ TaskRow... ]` (newest-first) | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 TaskRow` / `404` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTaskRequest (request body)

```json
{
  "userQuery": "Get the current weather and 5-day forecast for London",
  "submittedBy": "operator-42"
}
```

### TaskRow (response body)

```json
{
  "taskId": "t-9c1...",
  "request": {
    "taskId": "t-9c1...",
    "userQuery": "Get the current weather and 5-day forecast for London",
    "submittedBy": "operator-42",
    "submittedAt": "2026-06-29T08:10:00Z"
  },
  "discovered": {
    "tools": [
      {
        "toolId": "current-weather",
        "name": "Current Weather",
        "description": "Returns current temperature, conditions, and wind speed for a named city.",
        "category": "weather",
        "schemaJson": "{ \"type\": \"object\", \"properties\": { \"city\": { \"type\": \"string\" } }, \"required\": [\"city\"] }"
      },
      {
        "toolId": "forecast-5day",
        "name": "5-Day Forecast",
        "description": "Returns a 5-day hourly forecast for a named city.",
        "category": "weather",
        "schemaJson": "{ \"type\": \"object\", \"properties\": { \"city\": { \"type\": \"string\" } }, \"required\": [\"city\"] }"
      }
    ],
    "discoveredAt": "2026-06-29T08:10:01Z"
  },
  "result": {
    "output": "London: currently 14 °C, overcast. Forecast: Mon 15 °C / rain, Tue 17 °C / cloudy, ...",
    "toolsUsed": ["current-weather", "forecast-5day"],
    "guardedTools": [],
    "completedAt": "2026-06-29T08:10:22Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-29T08:10:00Z",
  "finishedAt": "2026-06-29T08:10:22Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: task-update
data: { "taskId": "t-9c1...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`RECEIVED`, `DISCOVERING`, `READY`, `EXECUTING`, `COMPLETED`, `FAILED`). Clients reconcile by `taskId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
