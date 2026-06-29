# API contract — triage-router-ui

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/messages` | — | `200 [ Message... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `TriageEndpoint` ← `MessageView` |
| `GET` | `/api/messages/{id}` | — | `200 Message` / `404` | `TriageEndpoint` ← `MessageView` |
| `POST` | `/api/messages` | `{ "customerId": String, "channel": String, "subject": String, "body": String }` | `201 { "messageId": String }` | `TriageEndpoint` → `MessageQueue` |
| `POST` | `/api/messages/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Message` (status now `RESOLVED`) / `409` if not `BLOCKED` | `TriageEndpoint` → `MessageEntity` |
| `GET` | `/api/messages/sse` | — | `text/event-stream` | `TriageEndpoint` ← `MessageView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### SupportMessage (request body for `POST /api/messages`, minus `messageId` and `receivedAt`)

```json
{
  "customerId": "cust-441",
  "channel": "chat",
  "subject": "Double charge on last invoice",
  "body": "Hi, I see two identical line items for the monthly plan on my October invoice. Can you remove the duplicate?"
}
```

### Message

```json
{
  "messageId": "msg-7c4f92a1",
  "incoming": {
    "messageId": "msg-7c4f92a1",
    "customerId": "cust-441",
    "channel": "chat",
    "subject": "Double charge on last invoice",
    "body": "Hi, I see two identical line items for the monthly plan on my October invoice. Can you remove the duplicate?",
    "receivedAt": "2026-06-28T09:12:00Z"
  },
  "route": {
    "category": "BILLING",
    "confidence": "high",
    "reason": "Explicit duplicate-charge complaint on a subscription invoice."
  },
  "reply": {
    "replySubject": "Re: Double charge on last invoice",
    "replyBody": "Thank you for reaching out. We can see the duplicate line item on your October invoice. We have initiated the removal process...",
    "action": "REFUND_INITIATED",
    "handlerTag": "billing",
    "repliedAt": "2026-06-28T09:12:14Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": "2026-06-28T09:12:15Z"
}
```

A blocked message has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated message has `route.category = "UNCLEAR"` (or workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### RouteDecision (returned by `RouterAgent`, embedded in `Message.route`)

```json
{ "category": "BILLING" | "PRODUCT" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### HandlerReply (returned by either handler, embedded in `Message.reply`)

```json
{ "replySubject": "Re: …",
  "replyBody": "3–5 short paragraphs",
  "action": "REFUND_INITIATED" | "ARTICLE_LINKED" | "FOLLOW_UP_SCHEDULED" | "INFO_PROVIDED" | "ESCALATED",
  "handlerTag": "billing" | "product",
  "repliedAt": "ISO-8601" }
```

### GuardrailResult (returned by `DraftGuardrail`, embedded in `Message.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-refund-timeline", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

## SSE event format

```
event: message-update
data: { "messageId": "msg-…", "status": "ROUTED_BILLING", … full Message JSON … }
```

One event per state transition on the `MessageView`. Clients reconcile by `messageId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/messages`).
- `404` — unknown `messageId`.
- `409` — `unblock` requested on a message not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
