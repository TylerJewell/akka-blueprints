# API contract — fomc-event-analyst

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/analyses` | `SubmitAnalysisRequest` | `201 { analysisId }` | `FomcEndpoint` → `FomcEventEntity` |
| `GET` | `/api/analyses` | — | `200 [ AnalysisRecord... ]` (newest-first) | `FomcEndpoint` ← `PolicyAnalysisView` |
| `GET` | `/api/analyses/{id}` | — | `200 AnalysisRecord` / `404` | `FomcEndpoint` ← `PolicyAnalysisView` |
| `GET` | `/api/analyses/sse` | — | `text/event-stream` | `FomcEndpoint` ← `PolicyAnalysisView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAnalysisRequest (request body)

```json
{
  "eventId": "2026-Q2-rate-decision"
}
```

### AnalysisRecord (response body)

```json
{
  "analysisId": "a-7bc3d9...",
  "eventId": "2026-Q2-rate-decision",
  "snapshot": {
    "eventId": "2026-Q2-rate-decision",
    "indicators": [
      {
        "indicatorId": "cpi-yoy-2026-06",
        "name": "CPI year-over-year",
        "value": 2.4,
        "unit": "percent",
        "observedAt": "2026-06-01T00:00:00Z"
      }
    ],
    "yieldCurve": {
      "twoYear": 4.35,
      "tenYear": 4.52,
      "spread": 0.17,
      "snapshotAt": "2026-06-10T14:00:00Z"
    },
    "gatheredAt": "2026-06-10T14:00:00Z"
  },
  "signalSet": {
    "signals": [
      {
        "signalId": "ps-a1b2c3d4",
        "direction": "hawkish",
        "magnitude": "moderate",
        "rationale": "CPI remaining above target at 2.4% provides grounds for continued rate firmness.",
        "groundedIndicatorId": "cpi-yoy-2026-06"
      }
    ],
    "forecast": {
      "forecastId": "fc-2026-q2",
      "basisPoints": 0,
      "confidence": 0.62,
      "narrative": "Mixed signals point to a hold decision."
    },
    "interpretedAt": "2026-06-10T14:00:05Z"
  },
  "analysis": {
    "eventName": "FOMC June 2026 rate decision",
    "rateOutlook": "hold",
    "executiveSummary": "The June 2026 FOMC meeting is expected to hold rates ...",
    "sections": [
      {
        "signalId": "ps-a1b2c3d4",
        "heading": "Inflation sustains hawkish pressure",
        "body": "CPI at 2.4% year-over-year keeps inflation above the 2% target ...",
        "groundedIn": [
          {
            "indicatorId": "cpi-yoy-2026-06",
            "indicatorName": "CPI year-over-year",
            "value": 2.4,
            "unit": "percent"
          }
        ]
      }
    ],
    "forecast": {
      "forecastId": "fc-2026-q2",
      "basisPoints": 0,
      "confidence": 0.62,
      "narrative": "Mixed signals point to a hold decision."
    },
    "synthesizedAt": "2026-06-10T14:00:10Z"
  },
  "review": {
    "verdict": "ACCEPTED",
    "reason": "All quality checks passed: signal attribution, market-indicator grounding, and non-empty sections verified.",
    "reviewedAt": "2026-06-10T14:00:11Z"
  },
  "status": "REVIEWED",
  "createdAt": "2026-06-10T14:00:00Z",
  "finishedAt": "2026-06-10T14:00:11Z",
  "guardrailRejections": []
}
```

An analysis in which the guardrail fired on the first SYNTHESIZE attempt:

```json
{
  "analysisId": "a-3ef1c8...",
  "status": "REVIEWED",
  "review": {
    "verdict": "ACCEPTED",
    "reason": "All quality checks passed on retry.",
    "reviewedAt": "2026-06-10T14:01:05Z"
  },
  "guardrailRejections": [
    {
      "phase": "SYNTHESIZE",
      "reason": "financial-quality-violation: market-grounding: section 'ps-a1b2c3d4' references indicatorId 'pcr-deflator-2026-06' not found in recorded MarketSnapshot",
      "rejectedAt": "2026-06-10T14:00:58Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the guardrail fired.

### SSE event format

```
event: analysis-update
data: { "analysisId": "a-7bc3d9...", "status": "INTERPRETED", "signalSet": { ... }, ... }
```

One event per state transition (`CREATED`, `GATHERING`, `GATHERED`, `INTERPRETING`, `INTERPRETED`, `SYNTHESIZING`, `SYNTHESIZED`, `REVIEWED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: analysis-rejection
data: { "analysisId": "a-7bc3d9...", "phase": "SYNTHESIZE", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `analysisId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitAnalysisRequest` record and the `AnalysisCreated` event to carry it.
