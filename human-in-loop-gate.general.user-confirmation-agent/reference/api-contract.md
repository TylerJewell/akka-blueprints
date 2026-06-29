# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/confirmation-request` | `{ "description": "string" }` | `{ "requestId": "uuid" }` | ConfirmationEndpoint → ConfirmationWorkflow |
| POST | `/api/requests/{requestId}/confirm` | `{ "confirmedBy": "string", "note": "string" }` | `200` \| `404` | ConfirmationEndpoint → RequestEntity |
| POST | `/api/requests/{requestId}/cancel` | `{ "cancelledBy": "string", "reason": "string" }` | `200` \| `404` | ConfirmationEndpoint → RequestEntity |
| GET | `/api/requests` | — | `{ "requests": [Request, ...] }` | ConfirmationEndpoint → RequestsView |
| GET | `/api/requests/{requestId}` | — | `Request` \| `404` | ConfirmationEndpoint → RequestEntity |
| GET | `/api/requests/sse` | — | SSE stream of `Request` | ConfirmationEndpoint → RequestsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ConfirmationEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ConfirmationEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ConfirmationEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Request` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "description": "string or null",
  "status": "PENDING_CONFIRMATION | CONFIRMED | CANCELLED | EXECUTED",
  "plannedAt": "ISO-8601 or null",
  "proposedActions": ["string", ...],
  "confirmedAt": "ISO-8601 or null",
  "confirmedBy": "string or null",
  "confirmationNote": "string or null",
  "cancelledAt": "ISO-8601 or null",
  "cancelledBy": "string or null",
  "cancelReason": "string or null",
  "executedAt": "ISO-8601 or null",
  "executionSummary": "string or null"
}
```

## SSE event format

`GET /api/requests/sse` streams the `RequestsView` rows. Each message:

```
event: request
data: { ...Request... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
