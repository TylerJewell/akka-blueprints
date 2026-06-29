# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/ticket-request` | `{ "customerMessage": "string" }` | `{ "ticketId": "uuid" }` | SupportEndpoint → SupportWorkflow |
| POST | `/api/tickets/{ticketId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | SupportEndpoint → TicketEntity |
| POST | `/api/tickets/{ticketId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | SupportEndpoint → TicketEntity |
| GET | `/api/tickets` | — | `{ "tickets": [Ticket, ...] }` | SupportEndpoint → TicketsView |
| GET | `/api/tickets/{ticketId}` | — | `Ticket` \| `404` | SupportEndpoint → TicketEntity |
| GET | `/api/tickets/sse` | — | SSE stream of `Ticket` | SupportEndpoint → TicketsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | SupportEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | SupportEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | SupportEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Ticket` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "customerMessage": "string or null",
  "status": "TRIAGED | APPROVED | REJECTED | RESOLVED",
  "triagedAt": "ISO-8601 or null",
  "responseSummary": "string or null",
  "responseBody": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "resolvedAt": "ISO-8601 or null",
  "confirmationId": "string or null"
}
```

## SSE event format

`GET /api/tickets/sse` streams the `TicketsView` rows. Each message:

```
event: ticket
data: { ...Ticket... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
