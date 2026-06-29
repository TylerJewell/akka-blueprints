# API contract — hvac-analytics

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
  "question": "Is Zone-A running above its cooling setpoint?",
  "zoneScope": "Zone-A",
  "submittedBy": "facilities-engineer-07"
}
```

### Query (response body)

```json
{
  "queryId": "q-3ac...",
  "request": {
    "queryId": "q-3ac...",
    "question": "Is Zone-A running above its cooling setpoint?",
    "zoneScope": "Zone-A",
    "submittedBy": "facilities-engineer-07",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "snapshot": {
    "points": [
      {
        "metric": "supply-air-temp",
        "value": 75.2,
        "unit": "°F",
        "zone": "Zone-A",
        "recordedAt": "2026-06-28T10:00:00Z"
      },
      {
        "metric": "return-air-temp",
        "value": 77.8,
        "unit": "°F",
        "zone": "Zone-A",
        "recordedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "assembledAt": "2026-06-28T10:00:01Z",
    "zoneScope": "Zone-A"
  },
  "answer": {
    "assessment": "Zone-A is running 3.2°F above its nominal cooling setpoint of 72°F. Return-air-temp confirms the zone is accumulating heat.",
    "dataPoints": [
      {
        "metric": "supply-air-temp",
        "value": 75.2,
        "unit": "°F",
        "zone": "Zone-A",
        "recordedAt": "2026-06-28T10:00:00Z",
        "significance": "Supply air at 75.2°F is 3.2°F above the 72°F cooling setpoint, indicating inadequate cooling delivery."
      }
    ],
    "trend": "DEGRADING",
    "recommendedAction": "Inspect the Zone-A AHU cooling coil and refrigerant charge.",
    "answeredAt": "2026-06-28T10:00:28Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Data points are present with significance notes; recommended action begins with an actionable verb.",
    "evaluatedAt": "2026-06-28T10:00:29Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:29Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: query-update
data: { "queryId": "q-3ac...", "status": "ANSWER_RECORDED", "answer": { ... }, ... }
```

One event per state transition (`INITIATED`, `SNAPSHOT_READY`, `ANALYSING`, `ANSWER_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `queryId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
