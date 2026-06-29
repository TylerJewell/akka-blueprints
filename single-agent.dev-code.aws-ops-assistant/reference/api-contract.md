# API contract — aws-ops-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/ops-requests` | `SubmitOpsRequest` | `201 { opsRequestId }` | `OpsEndpoint` → `OpsRequestEntity` |
| `GET` | `/api/ops-requests` | — | `200 [ OpsRequestRow... ]` (newest-first) | `OpsEndpoint` ← `OpsView` |
| `GET` | `/api/ops-requests/{id}` | — | `200 OpsRequestRow` / `404` | `OpsEndpoint` ← `OpsView` |
| `POST` | `/api/ops-requests/{id}/confirm` | `ConfirmActionRequest` | `204` | `OpsEndpoint` → `OpsRequestEntity` |
| `POST` | `/api/ops-requests/{id}/halt` | `HaltRequest` | `204` | `OpsEndpoint` → `OpsRequestEntity` |
| `GET` | `/api/ops-requests/sse` | — | `text/event-stream` | `OpsEndpoint` ← `OpsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitOpsRequest (request body)

```json
{
  "requestText": "Resize instance i-0abc123 from t3.medium to t3.large",
  "context": "env=production, account=acme-prod",
  "scope": "MUTATING",
  "submittedBy": "sre-operator-7"
}
```

### ConfirmActionRequest (request body)

```json
{
  "actionId": "a-002",
  "outcome": "APPROVED",
  "decidedBy": "sre-operator-7"
}
```

### HaltRequest (request body)

```json
{
  "haltedBy": "sre-operator-7"
}
```

### OpsRequestRow (response body)

```json
{
  "opsRequestId": "ops-8f3...",
  "request": {
    "opsRequestId": "ops-8f3...",
    "requestText": "Resize instance i-0abc123 from t3.medium to t3.large",
    "context": "env=production, account=acme-prod",
    "scope": "MUTATING",
    "submittedBy": "sre-operator-7",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "pendingConfirmations": [
    {
      "actionId": "a-002",
      "action": {
        "actionId": "a-002",
        "awsService": "EC2",
        "apiCall": "ModifyInstanceAttribute",
        "resourceArn": "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123",
        "kind": "MUTATING",
        "rationale": "Resize instance i-0abc123 to the requested type."
      },
      "requestedAt": "2026-06-28T14:00:08Z"
    }
  ],
  "decisions": [],
  "report": null,
  "status": "AWAITING_CONFIRMATION",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": null
}
```

Once the report has landed:

```json
{
  "opsRequestId": "ops-8f3...",
  "request": { "...": "..." },
  "pendingConfirmations": [],
  "decisions": [
    {
      "actionId": "a-002",
      "outcome": "APPROVED",
      "decidedBy": "sre-operator-7",
      "decidedAt": "2026-06-28T14:00:22Z"
    }
  ],
  "report": {
    "status": "COMPLETED",
    "summary": "Described i-0abc123 (t3.medium, running). Modified instance type to t3.large after operator approval.",
    "actions": [
      {
        "actionId": "a-001",
        "action": {
          "actionId": "a-001",
          "awsService": "EC2",
          "apiCall": "DescribeInstances",
          "resourceArn": "*",
          "kind": "READ",
          "rationale": "Confirm current instance type before resize."
        },
        "outcome": "COMPLETED",
        "responseSnippet": "{\"Reservations\":[{\"Instances\":[{\"InstanceId\":\"i-0abc123\",\"InstanceType\":\"t3.medium\",\"State\":{\"Name\":\"running\"}}]}]}",
        "durationMs": 287
      },
      {
        "actionId": "a-002",
        "action": {
          "actionId": "a-002",
          "awsService": "EC2",
          "apiCall": "ModifyInstanceAttribute",
          "resourceArn": "arn:aws:ec2:us-east-1:123456789012:instance/i-0abc123",
          "kind": "MUTATING",
          "rationale": "Resize instance i-0abc123 to the requested type."
        },
        "outcome": "COMPLETED",
        "responseSnippet": "{\"RequestId\":\"4f2a8d90-mock\",\"Return\":true}",
        "durationMs": 1142
      }
    ],
    "completedAt": "2026-06-28T14:00:32Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:32Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: ops-request-update
data: { "opsRequestId": "ops-8f3...", "status": "AWAITING_CONFIRMATION", "pendingConfirmations": [...], ... }
```

One event per state transition (`SUBMITTED`, `PLANNING`, `AWAITING_CONFIRMATION`, `EXECUTING`, `COMPLETED`, `HALTED`, `FAILED`). Clients reconcile by `opsRequestId`; each event carries the full row at the moment of transition so a late-joining client does not need to replay prior events.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and populate `submittedBy` / `decidedBy` / `haltedBy` from the authenticated principal rather than the request body.
