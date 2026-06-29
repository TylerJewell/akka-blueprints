# API contract — financial-advisor-pipeline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/advisories` | `SubmitAdvisoryRequest` | `201 { advisoryId }` | `AdvisoryEndpoint` → `AdvisoryEntity` |
| `GET` | `/api/advisories` | — | `200 [ AdvisoryRecord... ]` (newest-first) | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/advisories/{id}` | — | `200 AdvisoryRecord` / `404` | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/advisories/sse` | — | `text/event-stream` | `AdvisoryEndpoint` ← `AdvisoryView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AdvisoryEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AdvisoryEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AdvisoryEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAdvisoryRequest (request body)

```json
{
  "query": "retirement portfolio rebalancing for age 55"
}
```

### AdvisoryRecord (response body)

```json
{
  "advisoryId": "a-9d3fc1...",
  "query": "retirement portfolio rebalancing for age 55",
  "snapshot": {
    "dataPoints": [
      {
        "sector": "equities",
        "metric": "S&P-500-1yr-return",
        "value": "12.4%",
        "source": "market/equities.json",
        "capturedAt": "2026-06-28T10:00:00Z"
      },
      {
        "sector": "fixed-income",
        "metric": "10yr-treasury-yield",
        "value": "4.6%",
        "source": "market/fixed-income.json",
        "capturedAt": "2026-06-28T10:00:00Z"
      }
    ],
    "benchmarkIndex": "S&P 500",
    "benchmarkReturn": 12.4,
    "researchedAt": "2026-06-28T10:00:05Z"
  },
  "strategy": {
    "approach": "Balanced rebalancing toward fixed-income to reduce sequence-of-returns risk at age 55.",
    "allocations": [
      { "assetClass": "equities", "targetPct": 55, "rationale": "Equity momentum per S&P-500-1yr-return 12.4%." },
      { "assetClass": "fixed-income", "targetPct": 45, "rationale": "Treasury yield at 4.6% supports income." }
    ],
    "rationale": [
      { "itemId": "ri-01", "text": "Equity allocation reduced from typical age-55 baseline.", "supportingMetric": "S&P-500-1yr-return" }
    ],
    "definedAt": "2026-06-28T10:00:15Z"
  },
  "executionPlan": {
    "actions": [
      {
        "sequence": 1,
        "description": "Reduce equity exposure by trimming large-cap index fund holdings.",
        "timing": "within 30 days",
        "instruments": [{ "ticker": "SPY", "name": "SPDR S&P 500 ETF", "assetClass": "equities" }]
      },
      {
        "sequence": 2,
        "description": "Increase fixed-income exposure via intermediate-term bond fund.",
        "timing": "within 30 days",
        "instruments": [{ "ticker": "BND", "name": "Vanguard Total Bond Market ETF", "assetClass": "fixed-income" }]
      }
    ],
    "horizon": "3–6 months",
    "plannedAt": "2026-06-28T10:00:25Z"
  },
  "riskProfile": {
    "band": "MODERATE",
    "volatility": {
      "portfolioStdDev": 0.11,
      "beta": 0.82,
      "sharpeRatio": 1.14
    },
    "mitigations": [
      "Diversify across asset classes to limit single-sector drawdown.",
      "Maintain six-month cash reserve to avoid forced selling during downturns."
    ],
    "assessedAt": "2026-06-28T10:00:35Z"
  },
  "report": {
    "title": "Retirement portfolio rebalancing — age 55 analysis",
    "summary": "Market conditions support a gradual shift from equities to fixed-income. The suggested allocation reduces drawdown risk while capturing current yield levels. Risk band is MODERATE.",
    "strategy": { "...": "..." },
    "executionPlan": { "...": "..." },
    "riskProfile": { "...": "..." },
    "reportedAt": "2026-06-28T10:00:36Z"
  },
  "compliance": {
    "score": 5,
    "rationale": "Risk band present, mitigations listed, allocation breadth met, all rationale metrics traceable.",
    "evaluatedAt": "2026-06-28T10:00:37Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:37Z",
  "disclaimerLog": [
    {
      "advisoryId": "a-9d3fc1...",
      "disclaimerText": "This content is for educational purposes only and does not constitute licensed financial advice. Consult a qualified financial professional before making investment decisions.",
      "injectedAt": "2026-06-28T10:00:06Z"
    }
  ],
  "sanitizerLog": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `disclaimerLog` is an array — it has one entry per phase response on the happy path (four entries for a fully evaluated advisory). `sanitizerLog` is an array — empty on the clean path, populated when the sanitizer matched a prohibited pattern.

### SSE event format

```
event: advisory-update
data: { "advisoryId": "a-9d3fc1...", "status": "STRATEGIZED", "strategy": { ... }, ... }
```

One event per status transition (`CREATED`, `RESEARCHING`, `RESEARCHED`, `STRATEGIZING`, `STRATEGIZED`, `PLANNING`, `PLANNED`, `ASSESSING`, `EVALUATED`, `FAILED`) and one per governance side-event:

```
event: advisory-disclaimer
data: { "advisoryId": "a-9d3fc1...", "disclaimerText": "...", "injectedAt": "..." }

event: advisory-sanitizer
data: { "advisoryId": "a-9d3fc1...", "matchedPattern": "guaranteed return", "redactedFragment": "guaranteed return on your equity holdings", "firedAt": "..." }
```

Clients reconcile by `advisoryId`; a status event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend `SubmitAdvisoryRequest` and `AdvisoryCreated` to carry it.
