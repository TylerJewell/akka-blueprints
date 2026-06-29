# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/tool-requests` | `{ "goal": "string" }` | `{ "requestId": "uuid" }` | ApprovalEndpoint → ApprovalWorkflow |
| POST | `/api/tool-requests/{requestId}/approve` | `{ "approvedBy": "string", "approverNote": "string", "editedParameters": "string or null" }` | `200` \| `404` | ApprovalEndpoint → ToolRequestEntity |
| POST | `/api/tool-requests/{requestId}/reject` | `{ "rejectedBy": "string", "reason": "string" }` | `200` \| `404` | ApprovalEndpoint → ToolRequestEntity |
| GET | `/api/tool-requests` | — | `{ "requests": [ToolRequest, ...] }` | ApprovalEndpoint → ToolRequestsView |
| GET | `/api/tool-requests/{requestId}` | — | `ToolRequest` \| `404` | ApprovalEndpoint → ToolRequestEntity |
| GET | `/api/tool-requests/sse` | — | SSE stream of `ToolRequest` | ApprovalEndpoint → ToolRequestsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ApprovalEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ApprovalEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ApprovalEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`ToolRequest` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "goal": "string or null",
  "status": "PENDING_APPROVAL | APPROVED | REJECTED | EXECUTED",
  "plannedAt": "ISO-8601 or null",
  "toolName": "string or null",
  "parameters": "JSON object string or null",
  "rationale": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "editedParameters": "JSON object string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectedBy": "string or null",
  "rejectReason": "string or null",
  "executedAt": "ISO-8601 or null",
  "executionOutput": "string or null"
}
```

### Approve request body detail

`editedParameters` is optional. When present, it must be a valid JSON object string. The executor uses `editedParameters` in preference to the original `parameters` when both are present. When absent or `null`, the executor uses the planner's original `parameters`.

## SSE event format

`GET /api/tool-requests/sse` streams the `ToolRequestsView` rows. Each message:

```
event: tool-request
data: { ...ToolRequest... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
