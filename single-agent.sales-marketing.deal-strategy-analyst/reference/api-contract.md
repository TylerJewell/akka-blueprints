# API contract — deal-strategy-analyst

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/deals` | `SubmitDealRequest` | `201 { dealId }` | `DealEndpoint` → `DealEntity` |
| `GET` | `/api/deals` | — | `200 [ Deal... ]` (newest-first) | `DealEndpoint` ← `DealView` |
| `GET` | `/api/deals/{id}` | — | `200 Deal` / `404` | `DealEndpoint` ← `DealView` |
| `GET` | `/api/deals/sse` | — | `text/event-stream` | `DealEndpoint` ← `DealView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitDealRequest (request body)

```json
{
  "dealName": "Acme Corp — Enterprise Platform Evaluation",
  "dealType": "enterprise-new",
  "rawContext": "Call notes 2026-06-15: spoke with [REDACTED] who raised concerns about ...",
  "stakeholders": [
    {
      "role": "Economic Buyer",
      "concern": "Total cost of ownership over 3 years",
      "engagementLevel": "low"
    },
    {
      "role": "Champion",
      "concern": "Integration with existing data pipeline",
      "engagementLevel": "high"
    },
    {
      "role": "Legal",
      "concern": "Data residency requirements",
      "engagementLevel": "medium"
    }
  ],
  "submittedBy": "ae-sales-rep-42"
}
```

### Deal (response body)

```json
{
  "dealId": "d-7ca...",
  "context": {
    "dealId": "d-7ca...",
    "dealName": "Acme Corp — Enterprise Platform Evaluation",
    "dealType": "enterprise-new",
    "rawContext": "(raw context preserved for audit)",
    "stakeholders": [
      { "role": "Economic Buyer", "concern": "Total cost of ownership over 3 years", "engagementLevel": "low" }
    ],
    "submittedBy": "ae-sales-rep-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "sanitized": {
    "redactedContext": "Call notes 2026-06-15: spoke with [REDACTED-COMPANY] who raised concerns about ...",
    "piiCategoriesFound": ["email", "phone", "person-name"]
  },
  "recommendation": {
    "urgency": "HIGH",
    "executiveSummary": "The economic buyer has not responded since the pricing review. The champion is supportive but needs updated materials to advance internally. Legal requires a data-residency addendum before approving.",
    "nextSteps": [
      {
        "stepId": "re-engage-economic-buyer",
        "actionType": "SCHEDULE_CALL",
        "stakeholderRole": "Economic Buyer",
        "rationale": "No response in 12 days since the TCO summary was sent. A live call is required to surface objections before the evaluation window closes.",
        "suggestedDeadline": "within 2 business days",
        "priority": 1
      },
      {
        "stepId": "provide-data-residency-addendum",
        "actionType": "FOLLOW_UP_EMAIL",
        "stakeholderRole": "Legal",
        "rationale": "Legal explicitly asked for a data residency addendum in the last email thread. Unblocks contract approval.",
        "suggestedDeadline": "today",
        "priority": 2
      }
    ],
    "decidedAt": "2026-06-28T09:00:22Z"
  },
  "eval": {
    "score": 4,
    "rationale": "All next steps have deadlines and multi-sentence rationales; urgency is consistent with action types.",
    "evaluatedAt": "2026-06-28T09:00:23Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: deal-update
data: { "dealId": "d-7ca...", "status": "RECOMMENDATION_RECORDED", "recommendation": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `ANALYSING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `dealId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
