# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/reviews` | `{ "contractRef": "string", "contractText": "string", "submittedBy": "string" }` | `{ "reviewId": "uuid" }` | `ReviewEndpoint` → `ContractQueue` |
| GET | `/api/reviews` | — | `{ "reviews": [ContractReviewRow, ...] }` | `ReviewEndpoint` → `ReviewView` |
| GET | `/api/reviews?status=APPROVED` | — | filtered list (client-side filter) | `ReviewEndpoint` |
| GET | `/api/reviews/{id}` | — | `ContractReviewRow` or 404 | `ReviewEndpoint` |
| POST | `/api/reviews/{id}/approve` | `{ "lawyerNote": "string" }` | `{ "reviewId": "uuid", "status": "APPROVED" }` | `ReviewEndpoint` → `ContractReviewEntity` |
| POST | `/api/reviews/{id}/reject` | `{ "lawyerNote": "string" }` | `{ "reviewId": "uuid", "status": "REJECTED" }` | `ReviewEndpoint` → `ContractReviewEntity` |
| GET | `/api/reviews/sse` | — | `text/event-stream` of `ContractReviewRow` | `ReviewEndpoint` → `ReviewView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ReviewEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ReviewEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ReviewEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/reviews` request:

```json
{
  "contractRef": "NDA-2026-Acme-Corp",
  "contractText": "This Non-Disclosure Agreement is entered into as of...",
  "submittedBy": "alice@example.com"
}
```

`ContractReviewRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "reviewId": "uuid",
  "contractRef": "string",
  "submittedBy": "string",
  "status": "QUEUED | IN_REVIEW | AWAITING_APPROVAL | APPROVED | REJECTED | DEGRADED | BLOCKED",
  "clauseSummary": {
    "clauses": [
      { "clauseId": "cl-01", "clauseType": "indemnity", "excerpt": "...", "summary": "..." }
    ],
    "analysedAt": "ISO-8601"
  },
  "riskReport": {
    "flags": [
      { "clauseId": "cl-01", "riskLevel": "HIGH", "rationale": "..." }
    ],
    "overallRisk": "HIGH",
    "scoredAt": "ISO-8601"
  },
  "redlines": {
    "suggestions": [
      { "clauseId": "cl-01", "originalText": "...", "proposedText": "...", "justification": "..." }
    ],
    "draftedAt": "ISO-8601"
  },
  "reviewPackage": {
    "guardrailVerdict": "ok",
    "consolidatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "lawyerNote": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

`POST /api/reviews/{id}/approve` request:

```json
{ "lawyerNote": "Redlines accepted; IP assignment clause narrowed appropriately." }
```

`POST /api/reviews/{id}/reject` request:

```json
{ "lawyerNote": "Limitation of liability cap is still unacceptable. Requires renegotiation." }
```

## SSE event format

`GET /api/reviews/sse` emits one event per review change:

```
event: review
data: { ...ContractReviewRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `reviewId`. No polling.
