# API contract

All endpoints are JSON unless noted. ACL: open to localhost (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/action-request` | `{ "operationRequest": "string" }` | `{ "actionId": "uuid" }` | ApprovalEndpoint → ApprovalWorkflow |
| POST | `/api/actions/{actionId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | ApprovalEndpoint → ActionEntity |
| POST | `/api/actions/{actionId}/deny` | `{ "deniedBy": "string", "reason": "string" }` | `200` \| `404` | ApprovalEndpoint → ActionEntity |
| GET | `/api/actions` | — | `{ "actions": [Action, ...] }` | ApprovalEndpoint → ActionsView |
| GET | `/api/actions/{actionId}` | — | `Action` \| `404` | ApprovalEndpoint → ActionEntity |
| GET | `/api/actions/sse` | — | SSE stream of `Action` | ApprovalEndpoint → ActionsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ApprovalEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ApprovalEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ApprovalEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Action` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "operationRequest": "string or null",
  "status": "PROPOSED | APPROVED | DENIED | EXECUTED",
  "proposedAt": "ISO-8601 or null",
  "actionType": "string or null",
  "target": "string or null",
  "rationale": "string or null",
  "estimatedImpact": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "deniedAt": "ISO-8601 or null",
  "deniedBy": "string or null",
  "denyReason": "string or null",
  "executedAt": "ISO-8601 or null",
  "outcome": "string or null",
  "executionDetails": "string or null"
}
```

## SSE event format

`GET /api/actions/sse` streams the `ActionsView` rows. Each message:

```
event: action
data: { ...Action... }
```

The UI subscribes once and replaces the matching row by `id` on each event. Status transitions from `PROPOSED` → `APPROVED` → `EXECUTED` (or `DENIED`) arrive as individual SSE events; the UI updates the displayed card in place without a page refresh.

## Request examples

Submit an operation request:
```sh
curl -X POST http://localhost:9657/api/action-request \
  -H 'Content-Type: application/json' \
  -d '{"operationRequest": "scale web-frontend to 5 replicas"}'
```

Approve a proposal:
```sh
curl -X POST http://localhost:9657/api/actions/{actionId}/approve \
  -H 'Content-Type: application/json' \
  -d '{"approvedBy": "ops-lead@example.com", "note": "Looks correct. Proceed."}'
```

Deny a proposal:
```sh
curl -X POST http://localhost:9657/api/actions/{actionId}/deny \
  -H 'Content-Type: application/json' \
  -d '{"deniedBy": "ops-lead@example.com", "reason": "Target service is under maintenance freeze."}'
```
