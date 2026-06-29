# API Contract — Financial Model Builder

## Endpoint table

| Method | Path | Auth | Request body | Response |
|---|---|---|---|---|
| POST | `/api/models` | Analyst | `SubmitModelRequest` | `{ "modelId": "..." }` 201 |
| GET | `/api/models` | Analyst | — | `List<ModelRecord>` 200 |
| GET | `/api/models/{id}` | Analyst | — | `ModelRecord` 200 / 404 |
| POST | `/api/models/{id}/approve` | Analyst | `AnalystActionRequest` | 204 |
| POST | `/api/models/{id}/reject` | Analyst | `AnalystActionRequest` | 204 |
| GET | `/api/models/sse` | Analyst | — | SSE stream |
| GET | `/api/metadata/readme` | Public | — | README.md (text/plain) |
| GET | `/api/metadata/risk-survey` | Public | — | risk-survey.yaml (text/plain) |
| GET | `/api/metadata/eval-matrix` | Public | — | eval-matrix.yaml (text/plain) |
| GET | `/` | Public | — | 302 → `/app/index.html` |
| GET | `/app/*` | Public | — | Static assets |

---

## Payload examples

### SubmitModelRequest

```json
{
  "ticker": "AAPL",
  "period": "2025-Q4"
}
```

### AnalystActionRequest

```json
{
  "analystNote": "Reviewed gross margin assumptions against 10-Q. Derivations are supported. Approving."
}
```

### ModelRecord (full example)

```json
{
  "modelId": "MODEL-AAPL-2025-Q4-001",
  "ticker": "AAPL",
  "filingData": {
    "filingId": "AAPL-2025-Q4-10Q",
    "ticker": "AAPL",
    "period": "2025-Q4",
    "items": [
      {
        "name": "Revenue",
        "period": "2025-Q4",
        "value": 124300000000,
        "unit": "USD",
        "sourceRef": "10Q-2025Q4-AAPL-Revenue"
      },
      {
        "name": "CostOfGoodsSold",
        "period": "2025-Q4",
        "value": 70500000000,
        "unit": "USD",
        "sourceRef": "10Q-2025Q4-AAPL-COGS"
      }
    ],
    "extractedAt": "2026-01-15T10:00:00Z"
  },
  "model": {
    "modelId": "MODEL-AAPL-2025-Q4-001",
    "ticker": "AAPL",
    "period": "2025-Q4",
    "rows": [
      {
        "rowId": "ROW-001",
        "label": "Gross Margin",
        "period": "2025-Q4",
        "value": 0.432,
        "derivation": "10Q-2025Q4-AAPL-Revenue"
      }
    ],
    "assumptions": [
      {
        "key": "revenue-growth-rate",
        "description": "YoY revenue growth rate assumed for forward projection",
        "assumedValue": 0.08
      }
    ],
    "builtAt": "2026-01-15T10:05:00Z"
  },
  "validation": {
    "flags": [
      {
        "flagId": "FLAG-001",
        "severity": "WARNING",
        "description": "Revenue growth assumption (8%) exceeds trailing 4-quarter average (5.2%).",
        "affectedRow": "ROW-001"
      }
    ],
    "confidence": 74,
    "validatedAt": "2026-01-15T10:08:00Z"
  },
  "analystNote": "Reviewed gross margin assumptions against 10-Q. Approving.",
  "eval": {
    "score": 4,
    "rationale": "Row provenance OK. Assumption deviation within threshold (1 flag, no CRITICAL). Confidence 74 >= 60. No CRITICAL flags.",
    "evaluatedAt": "2026-01-15T10:30:00Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-01-15T09:58:00Z",
  "finishedAt": "2026-01-15T10:30:00Z"
}
```

### SSE event — model-update

```
event: model-update
data: {
  "modelId": "MODEL-AAPL-2025-Q4-001",
  "status": "PENDING_REVIEW",
  "ticker": "AAPL",
  "period": "2025-Q4",
  "updatedAt": "2026-01-15T10:08:05Z"
}
```

### SSE event — model-rejection (guardrail block)

```
event: model-rejection
data: {
  "modelId": "MODEL-AAPL-2025-Q4-002",
  "rejectedTool": "computeRatio",
  "currentPhase": "EXTRACT",
  "toolPhase": "BUILD",
  "occurredAt": "2026-01-15T11:00:12Z"
}
```

---

## Authorization note

This sample does not ship an authentication provider. In a production deployment, `ModelEndpoint` routes that mutate state (`POST /api/models`, `/approve`, `/reject`) should be protected with at minimum a bearer token or mTLS. The `analystNote` field on approve/reject can carry a user identity claim for audit purposes. The sample uses no role checks — add them in `ModelEndpoint` before exposing this service externally.
