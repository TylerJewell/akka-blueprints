# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/action-request` | `{ "taskDescription": "string" }` | `{ "actionId": "uuid" }` | GateEndpoint → GateWorkflow |
| POST | `/api/actions/{actionId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | GateEndpoint → ActionEntity |
| POST | `/api/actions/{actionId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | GateEndpoint → ActionEntity |
| GET | `/api/actions` | — | `{ "actions": [Action, ...] }` | GateEndpoint → ActionsView |
| GET | `/api/actions/{actionId}` | — | `Action` \| `404` | GateEndpoint → ActionEntity |
| GET | `/api/actions/sse` | — | SSE stream of `Action` | GateEndpoint → ActionsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | GateEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | GateEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | GateEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Action` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "taskDescription": "string or null",
  "status": "PROPOSED | APPROVED | REJECTED | EXECUTED",
  "proposedAt": "ISO-8601 or null",
  "proposalSummary": "string or null",
  "estimatedImpact": "string or null",
  "actionPayload": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "executedAt": "ISO-8601 or null",
  "outcome": "string or null"
}
```

## SSE event format

`GET /api/actions/sse` streams the `ActionsView` rows. Each message:

```
event: action
data: { ...Action... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
