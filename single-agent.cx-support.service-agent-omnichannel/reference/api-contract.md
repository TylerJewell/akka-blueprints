# API contract — service-agent-omnichannel

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/cases` | `IngestCaseRequest` | `201 { caseId }` | `CaseEndpoint` → `CaseEntity` |
| `GET` | `/api/cases` | — | `200 [ Case... ]` (newest-first) | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/cases/{id}` | — | `200 Case` / `404` | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `CaseEndpoint` ← `CaseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### IngestCaseRequest (request body)

```json
{
  "channel": "WEB",
  "customerId": "cust-88421",
  "rawMessage": "Hi, I was charged twice for my subscription this month. The second charge appeared on the 15th for £29.99. Can you look into this please?",
  "scenario": "billing-dispute"
}
```

### Case (response body)

```json
{
  "caseId": "c-3af...",
  "message": {
    "caseId": "c-3af...",
    "channel": "WEB",
    "customerId": "cust-88421",
    "rawMessage": "(raw text preserved for audit)",
    "scenario": "billing-dispute",
    "receivedAt": "2026-06-28T14:22:00Z"
  },
  "sanitized": {
    "redactedMessage": "Hi, I was charged twice for my subscription this month. The second charge appeared on the 15th for £29.99. Can you look into this please?",
    "piiCategoriesFound": []
  },
  "category": "BILLING",
  "reply": {
    "resolutionIntent": "NEEDS_FOLLOW_UP",
    "replyText": "Thank you for reaching out. I've opened a case for our billing team. Could you confirm the invoice number for the duplicate charge? That will help us resolve this faster.",
    "channelFormat": "chat-markdown",
    "crmWritesApplied": ["case.create:priority=NORMAL,summary=Billing dispute - duplicate charge"],
    "decidedAt": "2026-06-28T14:22:18Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Reply is channel-appropriate and asks a useful clarifying question; resolution intent is consistent with the scenario.",
    "evaluatedAt": "2026-06-28T14:22:19Z"
  },
  "status": "REPLIED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: case-update
data: { "caseId": "c-3af...", "status": "REPLIED", "reply": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `TRIAGING`, `HANDLING`, `REPLIED`, `ESCALATED`, `RESOLVED`, `EVALUATED`, `FAILED`). Clients reconcile by `caseId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `customerId` from the authenticated principal rather than the request body. Channel-origin verification (confirming a WhatsApp message actually arrived from WhatsApp) is a deployer-side concern outside this blueprint's scope.
