# API contract — weather-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/queries` | `SubmitQueryRequest` | `201 { questionId }` | `QueryEndpoint` → `QueryEntity` + `QueryWorkflow` |
| `GET` | `/api/queries` | — | `200 [ Query... ]` (newest-first) | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/{id}` | — | `200 Query` / `404` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/queries/sse` | — | `text/event-stream` | `QueryEndpoint` ← `QueryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `QueryEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `QueryEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `QueryEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitQueryRequest (request body)

```json
{
  "questionText": "What is the weather in Tokyo right now?",
  "units": "METRIC",
  "submittedBy": "user-42"
}
```

`units` is one of `METRIC`, `IMPERIAL`, or `STANDARD`.

### Query (response body — fully answered)

```json
{
  "questionId": "q-7a3f...",
  "question": {
    "questionId": "q-7a3f...",
    "questionText": "What is the weather in Tokyo right now?",
    "units": "METRIC",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "geocoding": {
    "displayName": "Tokyo, Japan",
    "latitude": 35.6762,
    "longitude": 139.6503
  },
  "answer": {
    "questionId": "q-7a3f...",
    "locationDisplay": "Tokyo, Japan",
    "latitude": 35.6762,
    "longitude": 139.6503,
    "units": "METRIC",
    "current": {
      "temperatureDegrees": 28.4,
      "feelsLikeDegrees": 31.1,
      "windSpeedKmh": 14.2,
      "humidityPercent": 72,
      "description": "partly cloudy",
      "iconCode": "02d"
    },
    "forecast": null,
    "narrative": "Tokyo is currently partly cloudy with a temperature of 28°C (feels like 31°C). Humidity is at 72% with light winds.",
    "answeredAt": "2026-06-28T09:00:12Z"
  },
  "toolCalls": [
    {
      "toolName": "geocode",
      "parameters": { "location": "Tokyo" },
      "outcome": "success",
      "calledAt": "2026-06-28T09:00:04Z"
    },
    {
      "toolName": "getWeather",
      "parameters": { "lat": "35.6762", "lon": "139.6503", "units": "metric", "forecastDays": "1" },
      "outcome": "success",
      "calledAt": "2026-06-28T09:00:08Z"
    }
  ],
  "status": "ANSWERED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:12Z",
  "failureReason": null
}
```

### Query (response body — failed)

```json
{
  "questionId": "q-9b1e...",
  "question": {
    "questionId": "q-9b1e...",
    "questionText": "What is the weather in Zyxwvut City?",
    "units": "METRIC",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T09:05:00Z"
  },
  "geocoding": null,
  "answer": null,
  "toolCalls": [
    {
      "toolName": "geocode",
      "parameters": { "location": "Zyxwvut City" },
      "outcome": "success",
      "calledAt": "2026-06-28T09:05:03Z"
    }
  ],
  "status": "FAILED",
  "createdAt": "2026-06-28T09:05:00Z",
  "finishedAt": "2026-06-28T09:05:06Z",
  "failureReason": "Geocoding returned no results for: Zyxwvut City"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: query-update
data: { "questionId": "q-7a3f...", "status": "ANSWERED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PROCESSING`, `ANSWERED`, `FAILED`). Additionally, one event fires for each `ToolCallRecorded` event on the entity, carrying the updated `toolCalls` list. Clients reconcile by `questionId`; an event always carries the full row at the moment of transition.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
