# API contract — support-multi-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/cases` | — | `200 [ Case... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `SupportEndpoint` ← `CaseView` |
| `GET` | `/api/cases/{id}` | — | `200 Case` / `404` | `SupportEndpoint` ← `CaseView` |
| `POST` | `/api/cases` | `{ "customerId": String, "channel": String, "subject": String, "body": String }` | `201 { "caseId": String }` | `SupportEndpoint` → `TicketQueue` |
| `POST` | `/api/cases/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Case` (status now `RESOLVED`) / `409` if not `BLOCKED` | `SupportEndpoint` → `CaseEntity` |
| `GET` | `/api/cases/sse` | — | `text/event-stream` | `SupportEndpoint` ← `CaseView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundCase (request body for `POST /api/cases`, minus `caseId` and `receivedAt`)

```json
{
  "customerId": "cust-442",
  "channel": "email",
  "subject": "Invoice shows double charge",
  "body": "Hi, I see two identical charges of $29 on my invoice this month. Email: ada@example.com. Account ACCT-7721."
}
```

### Case

```json
{
  "caseId": "cs-4a91d3f2",
  "incoming": {
    "caseId": "cs-4a91d3f2",
    "customerId": "cust-442",
    "channel": "email",
    "subject": "Invoice shows double charge",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T10:12:00Z"
  },
  "sanitized": {
    "redactedSubject": "Invoice shows double charge",
    "redactedBody": "Hi, I see two identical charges of $29 on my invoice this month. Email: [REDACTED-EMAIL]. Account [REDACTED-ACCOUNT-ID].",
    "piiCategoriesFound": ["email", "account-id"]
  },
  "routing": {
    "category": "BILLING",
    "confidence": "high",
    "reason": "Explicit duplicate-charge refund request."
  },
  "resolution": {
    "responseSubject": "Re: Invoice shows double charge",
    "responseBody": "Looking at the charge in question, the duplicate appears to have been caused by...",
    "action": "REFUND_INITIATED",
    "specialistTag": "billing",
    "resolvedAt": "2026-06-28T10:12:20Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T10:12:00Z",
  "finishedAt": "2026-06-28T10:12:21Z"
}
```

A blocked case has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated case has `routing.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### RoutingDecision (returned by `RouterAgent`, embedded in `Case.routing`)

```json
{ "category": "BILLING" | "TECHNICAL" | "ACCOUNT" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### Resolution (returned by any specialist, embedded in `Case.resolution`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "3–5 short paragraphs",
  "action": "REFUND_INITIATED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "INFO_PROVIDED" | "ACCESS_RESTORED" | "ESCALATED",
  "specialistTag": "billing" | "technical" | "account",
  "resolvedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `DraftGuardrail`, embedded in `Case.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-refund-timeline", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

## SSE event format

```
event: case-update
data: { "caseId": "cs-…", "status": "ROUTED_BILLING", … full Case JSON … }
```

One event per state transition on the `CaseView`. Clients reconcile by `caseId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/cases`).
- `404` — unknown `caseId`.
- `409` — `unblock` requested on a case not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
