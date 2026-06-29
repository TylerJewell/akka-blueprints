# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/deals` | `{ "address": "string", "propertyType": "string", "askingPriceDollars": number }` | `{ "dealId": "uuid" }` | `DealEndpoint` → `DealQueue` |
| GET | `/api/deals` | — | `{ "deals": [DealRow, ...] }` | `DealEndpoint` → `DealView` |
| GET | `/api/deals?status=RECOMMENDED` | — | filtered list (client-side filter) | `DealEndpoint` |
| GET | `/api/deals/{id}` | — | `DealRow` or 404 | `DealEndpoint` |
| GET | `/api/deals/sse` | — | `text/event-stream` of `DealRow` | `DealEndpoint` → `DealView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `DealEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `DealEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `DealEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/deals` request:

```json
{
  "address": "1427 Maple St, Austin, TX 78701",
  "propertyType": "multifamily",
  "askingPriceDollars": 875000
}
```

`DealRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "dealId": "uuid",
  "address": "string",
  "propertyType": "string",
  "askingPriceDollars": 875000,
  "status": "SCOPING | EVALUATING | RECOMMENDED | DEGRADED | BLOCKED | REJECTED",
  "marketReport": {
    "comparables": [
      { "address": "...", "salePriceDollars": 850000, "saleDate": "2025-11-03", "sqft": 2400 }
    ],
    "neighborhoodTrend": "appreciating",
    "estimatedMarketValueDollars": 882000,
    "gatheredAt": "ISO-8601"
  },
  "financialModel": {
    "grossRentalIncome": 72000,
    "operatingExpenses": 21600,
    "netOperatingIncome": 50400,
    "capRate": 5.76,
    "cashOnCashReturn": 7.23,
    "debtServiceCoverageRatio": 1.31,
    "modelledAt": "ISO-8601"
  },
  "recommendation": {
    "summary": "string (80–150 words)",
    "recommendationVerdict": "proceed | pass",
    "guardrailVerdict": "ok",
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

`GET /api/deals/sse` emits one event per deal change:

```
event: deal
data: { ...DealRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `dealId`. No polling.
