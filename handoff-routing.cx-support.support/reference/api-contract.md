# API contract — support

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/tickets` | — | `200 [ Ticket... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `SupportEndpoint` ← `TicketView` |
| `GET` | `/api/tickets/{id}` | — | `200 Ticket` / `404` | `SupportEndpoint` ← `TicketView` |
| `POST` | `/api/tickets` | `{ "customerId": String, "channel": String, "subject": String, "body": String }` | `201 { "ticketId": String }` | `SupportEndpoint` → `RequestQueue` |
| `POST` | `/api/tickets/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Ticket` (status now `RESOLVED`) / `409` if not `BLOCKED` | `SupportEndpoint` → `TicketEntity` |
| `GET` | `/api/tickets/sse` | — | `text/event-stream` | `SupportEndpoint` ← `TicketView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingRequest (request body for `POST /api/tickets`, minus `ticketId` and `receivedAt`)

```json
{
  "customerId": "cust-918",
  "channel": "email",
  "subject": "Charged twice for last month",
  "body": "Hi, I see two charges of $42.00 on my last invoice. Email: ada@example.com. Order ORD-771."
}
```

### Ticket

```json
{
  "ticketId": "tk-2bf83b1c",
  "incoming": {
    "ticketId": "tk-2bf83b1c",
    "customerId": "cust-918",
    "channel": "email",
    "subject": "Charged twice for last month",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T07:45:00Z"
  },
  "sanitized": {
    "redactedSubject": "Charged twice for last month",
    "redactedBody": "Hi, I see two charges of $42.00 on my last invoice. Email: [REDACTED-EMAIL]. Order [REDACTED-ORDER-ID].",
    "piiCategoriesFound": ["email", "order-id"]
  },
  "triage": {
    "category": "BILLING",
    "confidence": "high",
    "reason": "Explicit duplicate-charge refund request."
  },
  "resolution": {
    "responseSubject": "Re: Charged twice for last month",
    "responseBody": "Looking at your account, the duplicate charge is confirmed...",
    "action": "REFUND_INITIATED",
    "specialistTag": "billing",
    "resolvedAt": "2026-06-28T07:45:18Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "triageScore": {
    "score": 5,
    "rationale": "Category right; reason names the duplicate-charge signal directly.",
    "scoredAt": "2026-06-28T07:45:12Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T07:45:00Z",
  "finishedAt": "2026-06-28T07:45:19Z"
}
```

A blocked ticket has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated ticket has `triage.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### TriageDecision (returned by `TriageAgent`, embedded in `Ticket.triage`)

```json
{ "category": "BILLING" | "TECHNICAL" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### Resolution (returned by either specialist, embedded in `Ticket.resolution`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "3–5 short paragraphs",
  "action": "REFUND_INITIATED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "INFO_PROVIDED" | "ESCALATED",
  "specialistTag": "billing" | "technical",
  "resolvedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ResponseGuardrail`, embedded in `Ticket.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-refund-timeline", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

### TriageScore (returned by `TriageJudge`, embedded in `Ticket.triageScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: ticket-update
data: { "ticketId": "tk-…", "status": "TRIAGED", … full Ticket JSON … }
```

One event per state transition on the `TicketView`. Clients reconcile by `ticketId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/tickets`).
- `404` — unknown `ticketId`.
- `409` — `unblock` requested on a ticket not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
