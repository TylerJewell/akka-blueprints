# API contract — time-series-forecasting (java)

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/forecasts` | `SubmitForecastRequest` | `201 { forecastId }` | `ForecastEndpoint` → `ForecastEntity` |
| `GET` | `/api/forecasts` | — | `200 [ Forecast... ]` (newest-first) | `ForecastEndpoint` ← `ForecastView` |
| `GET` | `/api/forecasts/{id}` | — | `200 Forecast` / `404` | `ForecastEndpoint` ← `ForecastView` |
| `GET` | `/api/forecasts/sse` | — | `text/event-stream` | `ForecastEndpoint` ← `ForecastView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitForecastRequest (request body)

```json
{
  "seriesName": "Q3 2026 Daily Revenue (USD)",
  "historicalData": [
    { "date": "2026-04-01", "value": 138420.0 },
    { "date": "2026-04-02", "value": 141060.0 },
    { "date": "2026-04-03", "value": 139750.0 }
  ],
  "config": {
    "horizonDays": 14,
    "confidenceLevel": 0.95,
    "granularity": "daily"
  },
  "submittedBy": "analyst-07"
}
```

### Forecast (response body)

```json
{
  "forecastId": "f-3ae...",
  "submission": {
    "forecastId": "f-3ae...",
    "seriesName": "Q3 2026 Daily Revenue (USD)",
    "historicalData": "(90 SeriesPoint entries preserved for audit)",
    "config": { "horizonDays": 14, "confidenceLevel": 0.95, "granularity": "daily" },
    "submittedBy": "analyst-07",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "quality": {
    "rowCount": 90,
    "rangeStart": "2026-04-01",
    "rangeEnd": "2026-06-28",
    "missingValueCount": 0,
    "isStationary": true,
    "qualitySummary": "90-row daily series; no gaps; ADF approximation indicates stationarity."
  },
  "result": {
    "steps": [
      {
        "forecastDate": "2026-06-29",
        "pointForecast": 143200.0,
        "lowerBound": 137500.0,
        "upperBound": 148900.0,
        "stepQualityScore": 5
      },
      {
        "forecastDate": "2026-06-30",
        "pointForecast": 144600.0,
        "lowerBound": 136200.0,
        "upperBound": 153000.0,
        "stepQualityScore": 4
      }
    ],
    "summary": "Mild upward trend of ~650 USD/day; no weekly seasonality detected. Confidence intervals widen at the standard rate for a 95% level. No recent anomaly.",
    "completedAt": "2026-06-28T09:00:28Z"
  },
  "driftEval": {
    "driftStatus": "STABLE",
    "mape": 0.042,
    "rationale": "MAPE of 4.2% against the last 14-day actuals window is within the STABLE threshold.",
    "evaluatedAt": "2026-06-28T09:00:29Z"
  },
  "status": "DRIFT_EVALUATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:29Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: forecast-update
data: { "forecastId": "f-3ae...", "status": "FORECAST_COMPLETED", "result": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `VALIDATED`, `FORECASTING`, `FORECAST_COMPLETED`, `DRIFT_EVALUATED`, `FAILED`). Clients reconcile by `forecastId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
