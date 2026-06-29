# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/financial` | `{ "query": "string" }` | `{ "reportId": "uuid" }` | `FinancialEndpoint` → `QueryQueue` |
| GET | `/api/financial` | — | `{ "reports": [FinancialReportRow, ...] }` | `FinancialEndpoint` → `FinancialView` |
| GET | `/api/financial?status=PUBLISHED` | — | filtered list (client-side filter) | `FinancialEndpoint` |
| GET | `/api/financial/{id}` | — | `FinancialReportRow` or 404 | `FinancialEndpoint` |
| GET | `/api/financial/sse` | — | `text/event-stream` of `FinancialReportRow` | `FinancialEndpoint` → `FinancialView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `FinancialEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `FinancialEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `FinancialEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/financial` request:

```json
{ "query": "Analyse NVDA risk/return for a growth portfolio" }
```

`FinancialReportRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reportId": "uuid",
  "query": "string",
  "status": "DRAFTING | IN_REVIEW | PUBLISHED | DEGRADED | BLOCKED",
  "marketData": {
    "dataPoints": [
      { "ticker": "NVDA", "metric": "Q3 Revenue", "value": "$18.1B", "source": "Bloomberg 2025-Q3" }
    ],
    "gatheredAt": "ISO-8601"
  },
  "planningAssessment": {
    "riskProfile": "moderate-growth",
    "allocationGuidance": ["If volatility remains below 30-day average, maintain current weight."],
    "rationale": "string",
    "assessedAt": "ISO-8601"
  },
  "narrative": {
    "narrativeText": "string",
    "keyMessages": ["..."],
    "draftedAt": "ISO-8601"
  },
  "consolidated": {
    "executiveSummary": "string",
    "sanitizerVerdict": "clean",
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

`GET /api/financial/sse` emits one event per report change:

```
event: report
data: { ...FinancialReportRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reportId`. No polling.
