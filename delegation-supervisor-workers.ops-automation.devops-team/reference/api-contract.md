# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/ops/changes` | `{ "targetEnvironment": "string", "changeType": "string", "description": "string" }` | `{ "changeId": "uuid" }` | `OpsEndpoint` → `PipelineQueue` |
| GET | `/api/ops/changes` | — | `{ "changes": [ChangeRequestRow, ...] }` | `OpsEndpoint` → `OpsView` |
| GET | `/api/ops/changes?status=AWAITING_APPROVAL` | — | filtered list (client-side filter) | `OpsEndpoint` |
| GET | `/api/ops/changes/{id}` | — | `ChangeRequestRow` or 404 | `OpsEndpoint` |
| POST | `/api/ops/changes/{id}/approve` | `{ "approvedBy": "string" }` | `{ "changeId": "uuid", "status": "APPROVED" }` | `OpsEndpoint` → `ChangeRequestEntity`, resumes `OpsWorkflow` |
| POST | `/api/ops/changes/{id}/reject` | `{ "reason": "string" }` | `{ "changeId": "uuid", "status": "REJECTED" }` | `OpsEndpoint` → `ChangeRequestEntity`, resumes `OpsWorkflow` |
| POST | `/api/ops/halt` | `{ "reason": "string" }` | `{ "signalId": "uuid" }` | `OpsEndpoint` → `HaltSignalEntity` |
| GET | `/api/ops/changes/sse` | — | `text/event-stream` of `ChangeRequestRow` | `OpsEndpoint` → `OpsView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `OpsEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `OpsEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `OpsEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/ops/changes` request:

```json
{
  "targetEnvironment": "staging",
  "changeType": "deploy",
  "description": "Roll out order-service v2.4.1 to staging — adds circuit-breaker config."
}
```

`ChangeRequestRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "changeId": "uuid",
  "targetEnvironment": "staging",
  "changeType": "deploy",
  "description": "string",
  "requestedBy": "string",
  "status": "PLANNING | IN_PROGRESS | AWAITING_APPROVAL | APPROVED | REJECTED | DEGRADED | BLOCKED | HALTED",
  "workPlan": {
    "infraQuery": "string",
    "deployQuery": "string",
    "obsQuery": "string",
    "targetEnvironment": "string"
  },
  "infraAssessment": {
    "driftSummary": "string",
    "quotaWarnings": ["string"],
    "dependencyIssues": ["string"],
    "riskLevel": "LOW | MEDIUM | HIGH | CRITICAL",
    "assessedAt": "ISO-8601"
  },
  "deployAssessment": {
    "manifestSummary": "string",
    "rollbackAvailable": true,
    "canaryStatus": "GREEN | YELLOW | INACTIVE",
    "riskLevel": "LOW | MEDIUM | HIGH | CRITICAL",
    "assessedAt": "ISO-8601"
  },
  "obsAssessment": {
    "firingAlerts": ["string"],
    "sloBurnRate": 0.03,
    "onCallCovered": true,
    "riskLevel": "LOW | MEDIUM | HIGH | CRITICAL",
    "assessedAt": "ISO-8601"
  },
  "report": {
    "summary": "string",
    "riskLevel": "LOW | MEDIUM | HIGH | CRITICAL",
    "guardrailVerdict": "ok",
    "consolidatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "approvedBy": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

`POST /api/ops/changes/{id}/approve` request:

```json
{ "approvedBy": "ops-lead@example.com" }
```

`POST /api/ops/halt` request:

```json
{ "reason": "Cascading deploy failures detected across three services — stopping all in-flight changes." }
```

Response:

```json
{ "signalId": "uuid" }
```

## SSE event format

`GET /api/ops/changes/sse` emits one event per change-request state transition:

```
event: change
data: { ...ChangeRequestRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `changeId`. No polling.
