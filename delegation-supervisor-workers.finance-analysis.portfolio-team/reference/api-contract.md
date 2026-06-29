# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/portfolio` | `{ "portfolioId": "string", "sector": "string", "holdings": [...] }` | `{ "reportId": "uuid" }` or 400 if sector prohibited | `PortfolioEndpoint` → `HoldingsQueue` |
| GET | `/api/portfolio` | — | `{ "reports": [PortfolioReportRow, ...] }` | `PortfolioEndpoint` → `PortfolioView` |
| GET | `/api/portfolio?status=CONSOLIDATED` | — | filtered list (client-side filter) | `PortfolioEndpoint` |
| GET | `/api/portfolio/{id}` | — | `PortfolioReportRow` or 404 | `PortfolioEndpoint` |
| GET | `/api/portfolio/sse` | — | `text/event-stream` of `PortfolioReportRow` | `PortfolioEndpoint` → `PortfolioView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `PortfolioEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `PortfolioEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `PortfolioEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/portfolio` request:

```json
{
  "portfolioId": "port-001",
  "sector": "technology",
  "holdings": [
    { "ticker": "AAPL", "name": "Apple Inc.", "weight": 0.25, "marketValue": 250000.0 },
    { "ticker": "MSFT", "name": "Microsoft Corp.", "weight": 0.20, "marketValue": 200000.0 }
  ]
}
```

`POST /api/portfolio` 400 response (prohibited sector):

```json
{ "error": "sector 'weapons' is not permitted", "code": "SECTOR_PROHIBITED" }
```

`PortfolioReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "portfolioId": "port-001",
  "sector": "technology",
  "status": "PLANNING | IN_PROGRESS | CONSOLIDATED | DEGRADED | BLOCKED",
  "holdingsAssessment": {
    "notes": [
      { "ticker": "AAPL", "observation": "25% weight — high concentration in single name", "riskLevel": "MEDIUM" }
    ],
    "sectorExposure": "Large-cap technology, 90% allocation",
    "assessedAt": "ISO-8601"
  },
  "marketContext": {
    "macroSummary": "...",
    "sectorHeadwinds": ["rising input costs", "regulatory scrutiny on data practices"],
    "sectorTailwinds": ["cloud migration demand", "AI adoption tailwind"],
    "gatheredAt": "ISO-8601"
  },
  "consolidated": {
    "executive": "...",
    "guardrailVerdict": "ok",
    "consolidatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/portfolio/sse` emits one event per report change:

```
event: report
data: { ...PortfolioReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
