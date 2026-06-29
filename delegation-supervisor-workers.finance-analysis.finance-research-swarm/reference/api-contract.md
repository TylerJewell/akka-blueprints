# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/reports` | `{ "query": "string" }` | `{ "reportId": "uuid" }` | `ReportEndpoint` → `RequestQueue` |
| GET | `/api/reports` | — | `{ "reports": [StockReportRow, ...] }` | `ReportEndpoint` → `StockReportView` |
| GET | `/api/reports?status=PUBLISHED` | — | filtered list (client-side filter) | `ReportEndpoint` |
| GET | `/api/reports/{id}` | — | `StockReportRow` or 404 | `ReportEndpoint` |
| GET | `/api/reports/sse` | — | `text/event-stream` of `StockReportRow` | `ReportEndpoint` → `StockReportView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ReportEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ReportEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ReportEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/reports` request:

```json
{ "query": "Apple" }
```

`StockReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "query": "string",
  "ticker": {
    "ticker": "AAPL",
    "exchange": "NASDAQ",
    "companyName": "Apple Inc."
  },
  "status": "RESOLVING | IN_PROGRESS | PUBLISHED | PARTIAL | BLOCKED",
  "priceSummary": {
    "ticker": "AAPL",
    "recentPrices": [
      { "date": "2026-06-20", "open": 198.50, "close": 199.80, "high": 201.00, "low": 197.20, "volume": 54320000 }
    ],
    "changePercent": 1.4,
    "trend": "upward",
    "retrievedAt": "ISO-8601"
  },
  "newsSummary": {
    "ticker": "AAPL",
    "headlines": [
      { "headline": "...", "source": "Reuters", "publishedAt": "ISO-8601", "snippet": "..." }
    ],
    "sentimentLabel": "positive | negative | neutral",
    "retrievedAt": "ISO-8601"
  },
  "fundamentals": {
    "ticker": "AAPL",
    "peRatio": 28.5,
    "eps": 6.84,
    "marketCapUsd": 3040000000000,
    "revenueGrowthPercent": 4.2,
    "retrievedAt": "ISO-8601"
  },
  "integrated": {
    "summary": "string (80–150 words)",
    "sanitizerVerdict": "ok | blocked:<phrase>",
    "guardrailVerdict": "ok | blocked:<reason>",
    "integratedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/reports/sse` emits one event per report change:

```
event: report
data: { ...StockReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
