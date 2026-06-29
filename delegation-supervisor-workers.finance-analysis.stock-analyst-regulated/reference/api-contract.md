# API contract — Stock Analysis Team

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/analyses` | `{ "ticker": "ACME" }` | `{ "analysisId": "uuid" }` | StockAnalysisEndpoint → AnalysisWorkflow |
| POST | `/api/analyses/{id}/clear` | `{ "reviewer": "string", "note": "string" }` | `200` \| `404` | StockAnalysisEndpoint → AnalysisEntity |
| POST | `/api/analyses/{id}/retract` | `{ "reviewer": "string", "note": "string" }` | `200` \| `404` | StockAnalysisEndpoint → AnalysisEntity |
| GET | `/api/analyses?status=...` | — | `{ "analyses": [Analysis, ...] }` | StockAnalysisEndpoint → AnalysisView (filtered client-side) |
| GET | `/api/analyses/{id}` | — | `Analysis` | StockAnalysisEndpoint → AnalysisView |
| GET | `/api/analyses/sse` | — | SSE stream of `Analysis` | StockAnalysisEndpoint → AnalysisView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | StockAnalysisEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | StockAnalysisEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | StockAnalysisEndpoint |
| GET | `/` | — | `302 → /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Analysis JSON form

Lifecycle fields are nullable (`Optional<T>` in Java; serialized as the raw value or `null`).

```json
{
  "id": "uuid",
  "ticker": "ACME",
  "status": "QUEUED | RESEARCHING | SUMMARIZING | SANITIZING | RECOMMENDING | ISSUED_PENDING_REVIEW | CLEARED | RETRACTED | FAILED",
  "newsFindings": { "headlines": ["..."], "sentiment": "positive | neutral | negative" },
  "filingFindings": { "revenue": "string", "margin": "string", "riskFactors": ["..."] },
  "summary": "string or null",
  "recommendation": "BUY | HOLD | SELL or null",
  "rationale": "string or null",
  "disclaimer": "string or null",
  "sanitizedNotes": "string or null",
  "driftFlag": "boolean or null",
  "reviewedBy": "string or null",
  "reviewNote": "string or null",
  "queuedAt": "ISO-8601 or null",
  "researchedAt": "ISO-8601 or null",
  "summarizedAt": "ISO-8601 or null",
  "sanitizedAt": "ISO-8601 or null",
  "issuedAt": "ISO-8601 or null",
  "reviewedAt": "ISO-8601 or null",
  "failedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/analyses/sse` emits one `data:` line per updated analysis, each carrying the full `Analysis` JSON above. The UI replaces its row for that `id` on each event.
