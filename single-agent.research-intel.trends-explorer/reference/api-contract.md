# API contract — google-trends-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/trends` | `SubmitTrendRequest` | `201 { requestId }` | `TrendEndpoint` → `TrendRequestEntity` |
| `GET` | `/api/trends` | — | `200 [ TrendRequest... ]` (newest-first) | `TrendEndpoint` ← `TrendReportView` |
| `GET` | `/api/trends/{id}` | — | `200 TrendRequest` / `404` | `TrendEndpoint` ← `TrendReportView` |
| `GET` | `/api/trends/sse` | — | `text/event-stream` | `TrendEndpoint` ← `TrendReportView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTrendRequest (request body)

```json
{
  "region": "US",
  "timeWindow": "PAST_WEEK",
  "category": "Technology",
  "submittedBy": "analyst-42"
}
```

`timeWindow` must be one of: `PAST_DAY`, `PAST_WEEK`, `PAST_MONTH`, `PAST_90_DAYS`.

`category` is free-text but must match a seeded category (`All`, `Technology`, `Finance`, `Health`, `Entertainment`) to resolve against seed data; unrecognised categories fall back to the `All` seed for the given region/window pair.

### TrendRequest (response body)

```json
{
  "requestId": "tr-9a3...",
  "query": {
    "requestId": "tr-9a3...",
    "region": "US",
    "timeWindow": "PAST_WEEK",
    "category": "Technology",
    "submittedBy": "analyst-42",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "rawPayload": {
    "region": "US",
    "timeWindow": "PAST_WEEK",
    "category": "Technology",
    "topics": [
      {
        "topicName": "Quantum Computing",
        "searchVolumeIndex": 87,
        "breakout": true,
        "relatedQueries": ["quantum supremacy", "IBM quantum", "qubit error rate"]
      }
    ],
    "fetchedAt": "2026-06-28T10:00:01Z"
  },
  "report": {
    "region": "US",
    "timeWindow": "PAST_WEEK",
    "category": "Technology",
    "topics": [
      {
        "rank": 1,
        "topicName": "Quantum Computing",
        "searchVolumeIndex": 87,
        "breakout": true,
        "rationale": "Highest search volume this week and a breakout signal; related queries centre on IBM quantum-error-correction milestone coverage."
      }
    ],
    "breakoutSummary": "One breakout topic detected in US Technology this week: Quantum Computing exceeded the 5 000% growth threshold, driven by IBM's error-correction announcement and follow-on media coverage.",
    "clusters": [
      {
        "anchorTerm": "Quantum Computing",
        "relatedQueries": ["quantum supremacy", "IBM quantum", "qubit error rate"]
      }
    ],
    "reportedAt": "2026-06-28T10:00:18Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Every topic has a substantive rationale, breakout summary is present, and all cluster anchors match topic names.",
    "evaluatedAt": "2026-06-28T10:00:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: trend-update
data: { "requestId": "tr-9a3...", "status": "REPORT_READY", "report": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `DATA_READY`, `SYNTHESIZING`, `REPORT_READY`, `EVALUATED`, `FAILED`). Clients reconcile by `requestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
