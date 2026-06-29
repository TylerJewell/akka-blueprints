# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/emails` | `EmailIn` | `{ "emailId": "uuid" }` | `ResponderEndpoint` |
| POST | `/api/emails/{id}/approve` | `{ "approvedBy": "string" }` | `200` \| `404` | `ResponderEndpoint` |
| POST | `/api/emails/{id}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | `ResponderEndpoint` |
| GET | `/api/emails?status=...` | — | `{ "emails": [Email, ...] }` | `ResponderEndpoint` |
| GET | `/api/emails/{id}` | — | `Email` | `ResponderEndpoint` |
| GET | `/api/emails/sse` | — | SSE stream of `Email` | `ResponderEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ResponderEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ResponderEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ResponderEndpoint` |
| GET | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

The `?status=` filter is applied client-side from `getAllEmails` (the view has no enum `WHERE` clause — Lesson 2).

## Payload shapes

`EmailIn` (ingest request):

```json
{ "fromAddress": "string", "subject": "string", "body": "string" }
```

`Email` (lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "fromAddress": "string",
  "subject": "string",
  "body": "string",
  "status": "RECEIVED | SANITIZED | DRAFTED | AWAITING_REVIEW | APPROVED | REJECTED | SENT | ESCALATED",
  "receivedAt": "ISO-8601 or null",
  "sanitizedBody": "string or null",
  "draftSubject": "string or null",
  "draftBody": "string or null",
  "sensitive": "boolean or null",
  "draftedAt": "ISO-8601 or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "sentAt": "ISO-8601 or null",
  "sentMessageId": "string or null",
  "escalatedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/emails/sse` streams the `EmailsView` rows. Each event:

```
event: email
data: { ...Email JSON as above... }
```

The UI subscribes with `EventSource`, upserts each row by `id`, and re-renders the list.
