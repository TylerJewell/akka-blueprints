# API contract — live-eval-harness

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/evals` | — | `200 [ EvalViewRow... ]` (sorted newest-first). Optional `?status=…&agentId=…`. | `EvalEndpoint` ← `EvalView` |
| `GET` | `/api/evals/{id}` | — | `200 EvalViewRow` / `404` | `EvalEndpoint` ← `EvalView` |
| `GET` | `/api/evals/drift` | — | `200 DriftSnapshot` | `EvalEndpoint` ← `DriftSnapshotEntity` |
| `GET` | `/api/evals/sse` | — | `text/event-stream` | `EvalEndpoint` ← `EvalView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### EvalViewRow

```json
{
  "decisionId": "dec-7a3c…",
  "agentId": "agent-pricing-v2",
  "modelVersion": "gpt-4o-2024-11",
  "sanitizedInput": "User requested quote for [STRIPPED] account, product tier standard.",
  "sanitizedOutput": "Quoted price: $149/mo. Discount eligibility: not applicable.",
  "strippedFields": ["userId"],
  "evalResult": {
    "decisionId": "dec-7a3c…",
    "dimensions": [
      { "name": "accuracy",     "score": 4, "justification": "Price matches catalog for standard tier." },
      { "name": "consistency",  "score": 5, "justification": "Consistent with prior quotes in window." },
      { "name": "fairness",     "score": 4, "justification": "No visible subgroup bias in output." },
      { "name": "groundedness", "score": 3, "justification": "Discount eligibility not fully traced to input." }
    ],
    "overallScore": 4,
    "summary": "Mostly grounded; discount eligibility reasoning could be tighter.",
    "evaluatedAt": "2026-06-28T09:12:05Z"
  },
  "alarm": null,
  "status": "OK",
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": "2026-06-28T09:12:05Z"
}
```

### DriftSnapshot

```json
{
  "status": "WATCH",
  "narrative": "Fairness mean at 2.7 over last 50 decisions; approaching alarm threshold.",
  "flaggedDimensions": ["fairness"],
  "lastAssessedAt": "2026-06-28T09:00:00Z",
  "alarmCount": 0
}
```

### SSE event format

```
event: eval-update
data: { "decisionId": "...", "status": "ALARMED", "overallScore": 2, ... }

event: drift-update
data: { "status": "WATCH", "narrative": "...", "flaggedDimensions": ["fairness"] }
```

One `eval-update` event per decision state transition. One `drift-update` event per `DriftSampler` tick. Clients reconcile `eval-update` by `decisionId`; `drift-update` replaces the single global drift panel.
