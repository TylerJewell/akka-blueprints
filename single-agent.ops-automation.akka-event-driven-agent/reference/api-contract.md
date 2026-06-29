# API contract — ambient-durable-agent-pubsub

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/requests/publish` | `WeatherRequestPayload` | `201 { requestId }` | `RequestEndpoint` → topic → `TopicConsumer` |
| `GET` | `/api/requests` | — | `200 [ Request... ]` (newest-first) | `RequestEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 Request` / `404` | `RequestEndpoint` ← `RequestView` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `RequestEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### WeatherRequestPayload (POST /api/requests/publish body)

```json
{
  "requestId": "wr-7f3a...",
  "location": "Miami Beach, FL",
  "requesterContact": "ops-team@example.com",
  "horizon": "DAILY_3D",
  "alertCategories": ["STORM", "FLOOD"],
  "receivedAt": "2026-06-28T14:00:00Z"
}
```

`requestId` is minted by `RequestEndpoint` if not supplied. `requesterContact` is optional but included when present in the originating system.

### Request (response body)

```json
{
  "requestId": "wr-7f3a...",
  "rawPayload": {
    "requestId": "wr-7f3a...",
    "location": "Miami Beach, FL",
    "requesterContact": "ops-team@example.com",
    "horizon": "DAILY_3D",
    "alertCategories": ["STORM", "FLOOD"],
    "receivedAt": "2026-06-28T14:00:00Z"
  },
  "validation": {
    "valid": true,
    "violationCodes": []
  },
  "sanitized": {
    "location": "Miami Beach, FL",
    "horizon": "DAILY_3D",
    "alertCategories": ["STORM", "FLOOD"],
    "piiCategoriesFound": ["email"]
  },
  "report": {
    "conditionsSummary": "Tropical moisture will keep humidity high over the next three days with afternoon thunderstorm development likely each day.",
    "temperatureRange": { "lowCelsius": 26, "highCelsius": 33 },
    "windSpeedKph": 55,
    "precipitationPct": 85,
    "hazardFlag": true,
    "hazardDetail": "STORM: tropical thunderstorm development likely day 2–3 with gusts to 55 kph.",
    "forecastedAt": "2026-06-28T14:00:18Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:20Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

A `FAILED` request carries a non-null `validation.violationCodes` list if the failure was a payload validation failure, or `null` validation if the failure occurred in a later step.

### Validation failure example

```json
{
  "requestId": "wr-4c2b...",
  "rawPayload": {
    "requestId": "wr-4c2b...",
    "location": "",
    "requesterContact": null,
    "horizon": "DAILY_3D",
    "alertCategories": [],
    "receivedAt": "2026-06-28T14:01:00Z"
  },
  "validation": {
    "valid": false,
    "violationCodes": ["location.required"]
  },
  "sanitized": null,
  "report": null,
  "status": "FAILED",
  "createdAt": "2026-06-28T14:01:00Z",
  "finishedAt": "2026-06-28T14:01:00Z"
}
```

### SSE event format

```
event: request-update
data: { "requestId": "wr-7f3a...", "status": "COMPLETED", "report": { ... }, ... }
```

One event per state transition (`RECEIVED`, `VALIDATED`, `SANITIZED`, `FORECASTING`, `COMPLETED`, `FAILED`). Each event carries the full row at the moment of transition. A late-joining client never needs to replay — it receives the current state on connect.

## Violation codes (G1 guardrail)

| Code | Meaning |
|---|---|
| `location.required` | `location` is null or empty. |
| `location.too-long` | `location` exceeds 256 characters. |
| `horizon.invalid` | `horizon` is not a valid `ForecastHorizon` enum value. |
| `alertCategories.null` | `alertCategories` is null (empty list is accepted). |
| `requesterContact.injection` | `requesterContact` contains injection-pattern characters. |

## Authorization

ACL: open to the local network (local-dev only). A deployer adding identity should wrap `RequestEndpoint` with their auth middleware and populate `requesterContact` from the authenticated principal rather than the raw payload.
