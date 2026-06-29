# API contract — durable-weather-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `SubmitJobRequest` | `201 { jobId }` | `JobEndpoint` → `WeatherJobEntity` + `AgentWorkflow` |
| `GET` | `/api/jobs` | — | `200 [ WeatherJob... ]` (newest-first) | `JobEndpoint` ← `JobView` |
| `GET` | `/api/jobs/{id}` | — | `200 WeatherJob` / `404` | `JobEndpoint` ← `JobView` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` | `JobEndpoint` ← `JobView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitJobRequest (request body)

```json
{
  "queryText": "Current conditions and 3-day forecast for Berlin",
  "city": "Berlin",
  "mode": "TRIGGER_AGENT",
  "submittedBy": "user-42"
}
```

`mode` is `"TRIGGER_AGENT"` or `"CALL_AGENT"`. The `city` field is optional; if omitted the agent resolves it from `queryText`. `submittedBy` is a free-form caller identifier.

### WeatherJob (response body)

```json
{
  "jobId": "j-3af...",
  "query": {
    "jobId": "j-3af...",
    "queryText": "Current conditions and 3-day forecast for Berlin",
    "city": "Berlin",
    "mode": "TRIGGER_AGENT",
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "report": {
    "jobId": "j-3af...",
    "current": {
      "city": "Berlin",
      "country": "DE",
      "temperatureCelsius": 22.4,
      "conditionLabel": "Partly cloudy",
      "humidityPercent": 61,
      "windSpeedKph": 14.2,
      "observedAt": "2026-06-28T08:50:00Z"
    },
    "forecast": [
      {
        "date": "2026-06-29",
        "highCelsius": 24.0,
        "lowCelsius": 15.5,
        "conditionLabel": "Mostly sunny",
        "precipitationChancePercent": 10
      },
      {
        "date": "2026-06-30",
        "highCelsius": 20.1,
        "lowCelsius": 13.8,
        "conditionLabel": "Rain showers",
        "precipitationChancePercent": 75
      },
      {
        "date": "2026-07-01",
        "highCelsius": 18.6,
        "lowCelsius": 12.0,
        "conditionLabel": "Overcast",
        "precipitationChancePercent": 45
      }
    ],
    "activeAlerts": [],
    "narrativeSummary": "Berlin is currently 22 °C with partly cloudy skies and moderate humidity. Tomorrow looks pleasant with a high of 24 °C, but Tuesday brings rain showers with a 75% precipitation chance. No active weather alerts are in effect.",
    "toolCallLog": [
      {
        "toolName": "get_current_weather",
        "argumentsSummary": "city=Berlin",
        "status": "EXECUTED",
        "rawResponseSnippet": "{\"temp\": 22.4, \"condition\": \"Partly cloudy\", ...}",
        "guardRejectionReason": null,
        "calledAt": "2026-06-28T09:00:01Z"
      },
      {
        "toolName": "get_forecast",
        "argumentsSummary": "city=Berlin, days=3",
        "status": "EXECUTED",
        "rawResponseSnippet": "[{\"date\": \"2026-06-29\", ...}, ...]",
        "guardRejectionReason": null,
        "calledAt": "2026-06-28T09:00:02Z"
      }
    ],
    "completedAt": "2026-06-28T09:00:04Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:04Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

A `BLOCKED` tool call in `toolCallLog`:

```json
{
  "toolName": "get_forecast",
  "argumentsSummary": "city=Berlin, days=30",
  "status": "BLOCKED",
  "rawResponseSnippet": null,
  "guardRejectionReason": "days=30 exceeds maximum allowed horizon of 14",
  "calledAt": "2026-06-28T09:00:01Z"
}
```

### SSE event format

```
event: job-update
data: { "jobId": "j-3af...", "status": "COMPLETED", "report": { ... }, ... }
```

One event per state transition (`QUEUED`, `RUNNING`, `COMPLETED`, `FAILED`). Clients reconcile by `jobId`; an event always carries the full row at the moment of transition.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and supply `submittedBy` from the authenticated principal rather than the request body.
