# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/plan-request` | `{ "goal": "string" }` | `{ "planId": "uuid" }` | PlanEndpoint → PlanWorkflow |
| PATCH | `/api/plans/{planId}/edit` | `{ "editedBy": "string", "revisedSteps": ["string", ...] }` | `200` \| `404` | PlanEndpoint → PlanEntity |
| POST | `/api/plans/{planId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | PlanEndpoint → PlanEntity |
| POST | `/api/plans/{planId}/cancel` | `{ "cancelledBy": "string", "reason": "string" }` | `200` \| `404` | PlanEndpoint → PlanEntity |
| GET | `/api/plans` | — | `{ "plans": [Plan, ...] }` | PlanEndpoint → PlansView |
| GET | `/api/plans/{planId}` | — | `Plan` \| `404` | PlanEndpoint → PlanEntity |
| GET | `/api/plans/sse` | — | SSE stream of `Plan` | PlanEndpoint → PlansView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | PlanEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | PlanEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | PlanEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`Plan` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "goal": "string or null",
  "status": "PLANNED | APPROVED | CANCELLED | EXECUTED",
  "plannedAt": "ISO-8601 or null",
  "steps": ["string", "string"] ,
  "rationale": "string or null",
  "editedAt": "ISO-8601 or null",
  "editedBy": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "cancelledAt": "ISO-8601 or null",
  "cancelledBy": "string or null",
  "cancelReason": "string or null",
  "executedAt": "ISO-8601 or null",
  "outcome": "string or null"
}
```

`steps` is null before the plan is created; a list of one or more strings after `PlanCreated` or `PlanEdited`.

## SSE event format

`GET /api/plans/sse` streams the `PlansView` rows. Each message:

```
event: plan
data: { ...Plan... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
