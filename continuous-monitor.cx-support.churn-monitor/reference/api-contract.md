# API contract — churn-monitor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/churn` | — | `200 [ AccountChurnRow... ]` (sorted by riskLevel desc, then createdAt desc) | `ChurnEndpoint` ← `ChurnView` |
| `GET` | `/api/churn/{accountId}` | — | `200 AccountChurnRow` / `404` | `ChurnEndpoint` ← `ChurnView` |
| `POST` | `/api/churn/{accountId}/action` | `{ "decidedBy": String }` | `200 AccountChurnRow` (status now ACTIONED) | `ChurnEndpoint` → `AccountChurnEntity` |
| `POST` | `/api/churn/{accountId}/dismiss` | `{ "decidedBy": String, "reason": String }` | `200 AccountChurnRow` (status now DISMISSED) | `ChurnEndpoint` → `AccountChurnEntity` |
| `GET` | `/api/churn/sse` | — | `text/event-stream` | `ChurnEndpoint` ← `ChurnView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### AccountChurnRow (View row — no raw PII fields)

```json
{
  "accountId": "acct-a3f1…",
  "industry": "SaaS",
  "accountTier": "enterprise",
  "monthsActive": 14,
  "supportTicketsLast90d": 8,
  "usageRatioLast30d": 0.19,
  "piiCategoriesFound": ["email", "phone", "account-name"],
  "score": {
    "riskLevel": "HIGH",
    "probability": 0.87,
    "topSignals": ["usage ratio 0.19 (critical low)", "8 support tickets in 90d"]
  },
  "retentionPlan": {
    "actions": [
      { "actionType": "executive-outreach", "description": "Schedule a call with the account executive.", "priorityRank": 1 },
      { "actionType": "training-session", "description": "Offer a product deep-dive with a solutions engineer.", "priorityRank": 2 }
    ],
    "summary": "Prioritise executive engagement to understand usage barriers before offering discounts.",
    "advisedAt": "2026-06-28T09:15:03Z"
  },
  "decision": {
    "decidedBy": "csm-042",
    "reason": null,
    "decidedAt": "2026-06-28T10:02:17Z"
  },
  "latestEval": {
    "driftScore": 0.12,
    "driftVerdict": "Score distribution within expected range.",
    "fairnessScore": 0.88,
    "fairnessVerdict": "No significant inter-group disparity detected.",
    "evaluatedAt": "2026-06-28T12:00:05Z"
  },
  "status": "ACTIONED",
  "createdAt": "2026-06-28T09:14:50Z",
  "closedAt": null
}
```

### POST /api/churn/{accountId}/action body

```json
{ "decidedBy": "csm-042" }
```

### POST /api/churn/{accountId}/dismiss body

```json
{ "decidedBy": "csm-042", "reason": "Account confirmed renewal during offline call." }
```

### SSE event format

```
event: churn-update
data: { "accountId": "acct-a3f1…", "status": "ACTIONED", ... }
```

One event per state transition. Clients reconcile by `accountId`.
