# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/analysis` | `{ "ticker": "string" }` | `{ "reportId": "uuid" }` | `AnalysisEndpoint` → `TickerQueue` |
| GET | `/api/analysis` | — | `{ "reports": [StockReportRow, ...] }` | `AnalysisEndpoint` → `StockReportView` |
| GET | `/api/analysis?status=COMPLETE` | — | filtered list (client-side filter) | `AnalysisEndpoint` |
| GET | `/api/analysis/{id}` | — | `StockReportRow` or 404 | `AnalysisEndpoint` |
| GET | `/api/analysis/sse` | — | `text/event-stream` of `StockReportRow` | `AnalysisEndpoint` → `StockReportView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AnalysisEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AnalysisEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/analysis` request:

```json
{ "ticker": "AAPL" }
```

`StockReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "ticker": "AAPL",
  "status": "QUEUED | ANALYSING | COMPLETE | DEGRADED | BLOCKED",
  "financials": {
    "ticker": "AAPL",
    "revenueGrowthPct": 8.1,
    "epsLatest": 6.43,
    "operatingMarginPct": 31.5,
    "extractedAt": "ISO-8601"
  },
  "news": {
    "ticker": "AAPL",
    "items": [
      { "headline": "Apple reports record services revenue", "source": "Reuters", "sentiment": "positive" }
    ],
    "overallSentiment": "positive",
    "summarisedAt": "ISO-8601"
  },
  "ratios": {
    "ticker": "AAPL",
    "peRatio": 29.4,
    "pbRatio": 45.1,
    "roe": 147.2,
    "debtEquityRatio": 1.8,
    "computedAt": "ISO-8601"
  },
  "recommendation": {
    "ticker": "AAPL",
    "summary": "string (80-150 words)",
    "stance": "POSITIVE | NEUTRAL | CAUTIOUS | INSUFFICIENT_DATA",
    "disclaimerPresent": true,
    "sanitizerVerdict": "ok",
    "synthesisedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/analysis/sse` emits one event per report change:

```
event: report
data: { ...StockReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
