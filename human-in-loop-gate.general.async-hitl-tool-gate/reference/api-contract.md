# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Produced/consumed by `ActionEndpoint` except the static routes (`AppEndpoint`).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/requests` | `{ "request": "string" }` | `{ "actionId": "uuid" }` | ActionEndpoint |
| POST | `/api/actions/{id}/approve` | `{ "approvedBy": "string", "comment": "string" }` | `200` \| `404` | ActionEndpoint |
| POST | `/api/actions/{id}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | ActionEndpoint |
| GET | `/api/actions?status=...` | — | `{ "actions": [Action, ...] }` | ActionEndpoint |
| GET | `/api/actions/{id}` | — | `Action` \| `404` | ActionEndpoint |
| GET | `/api/actions/sse` | — | SSE stream of `Action` | ActionEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ActionEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ActionEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ActionEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static-resources file | AppEndpoint |

The `status` query param filters client-side over `ActionsView.getAllActions` — the view query carries no `WHERE status` clause (enum columns cannot be auto-indexed).

## Action JSON

Lifecycle fields are nullable (`Optional` in Java; serialized as raw value or `null`):

```json
{
  "id": "uuid",
  "request": "string or null",
  "status": "CREATED | PLANNED | APPROVED | REJECTED | EXECUTED | ESCALATED",
  "plannedAt": "ISO-8601 or null",
  "toolName": "string or null",
  "toolArguments": "string or null",
  "rationale": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverComment": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "executedAt": "ISO-8601 or null",
  "toolOutput": "string or null",
  "escalatedAt": "ISO-8601 or null"
}
```

## SSE format

`GET /api/actions/sse` streams `ActionsView` updates. Each event:

```
event: action
data: { ...Action JSON as above... }
```

The UI subscribes with `EventSource` and upserts each `Action` into the live list by `id`.
