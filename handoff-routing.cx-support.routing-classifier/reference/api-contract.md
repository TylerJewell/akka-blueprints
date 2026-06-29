# API contract — routing-classifier

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/cases` | — | `200 [ Case... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `RoutingEndpoint` ← `CaseView` |
| `GET` | `/api/cases/{id}` | — | `200 Case` / `404` | `RoutingEndpoint` ← `CaseView` |
| `POST` | `/api/cases` | `{ "customerId": String, "channel": String, "subject": String, "body": String }` | `201 { "caseId": String }` | `RoutingEndpoint` → `TicketQueue` |
| `POST` | `/api/cases/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Case` (status now `RESOLVED`) / `409` if not `BLOCKED` | `RoutingEndpoint` → `CaseEntity` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `RoutingEndpoint` ← `CaseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundTicket (request body for `POST /api/cases`, minus `caseId` and `receivedAt`)

```json
{
  "customerId": "cust-0042",
  "channel": "email",
  "subject": "Double charge on March invoice",
  "body": "I see two identical charges on my statement. My email is ada@example.com. Account ACCT-7712."
}
```

### Case

```json
{
  "caseId": "cs-4a91d7e2",
  "incoming": {
    "caseId": "cs-4a91d7e2",
    "customerId": "cust-0042",
    "channel": "email",
    "subject": "Double charge on March invoice",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedSubject": "Double charge on March invoice",
    "redactedBody": "I see two identical charges on my statement. My email is [REDACTED-EMAIL]. Account [REDACTED-ACCOUNT-ID].",
    "piiCategoriesFound": ["email", "account-id"]
  },
  "classification": {
    "category": "BILLING",
    "confidence": "high",
    "reason": "Explicit duplicate-charge complaint."
  },
  "reply": {
    "replySubject": "Re: Double charge on March invoice",
    "replyBody": "Thank you for reaching out. We've reviewed your account and confirmed the duplicate charge...",
    "action": "REFUND_INITIATED",
    "handlerTag": "billing",
    "repliedAt": "2026-06-28T09:15:14Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:15Z"
}
```

A blocked case has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated case has `classification.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### ClassificationResult (returned by `ClassifierAgent`, embedded in `Case.classification`)

```json
{ "category": "BILLING" | "TECHNICAL" | "ACCOUNT" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### HandlerReply (returned by any handler, embedded in `Case.reply`)

```json
{ "replySubject": "Re: …",
  "replyBody": "2–5 short paragraphs",
  "action": "REFUND_INITIATED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "INFO_PROVIDED" | "ACCOUNT_UPDATED" | "ESCALATED",
  "handlerTag": "billing" | "technical" | "account",
  "repliedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ResponseGuardrail`, embedded in `Case.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-resolution-timeline", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

## SSE event format

```
event: case-update
data: { "caseId": "cs-…", "status": "CLASSIFIED", … full Case JSON … }
```

One event per state transition on the `CaseView`. Clients reconcile by `caseId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/cases`).
- `404` — unknown `caseId`.
- `409` — `unblock` requested on a case not currently in `BLOCKED`.
- `503` — backend unavailable (workflow timed out, recovery in progress).
