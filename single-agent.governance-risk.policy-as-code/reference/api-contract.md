# API contract — policy-as-code

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/changes` | `SubmitChangeRequest` | `201 { changeId }` | `PolicyEndpoint` → `ChangeRequestEntity` |
| `GET` | `/api/changes` | — | `200 [ ChangeEvaluation... ]` (newest-first) | `PolicyEndpoint` ← `PolicyView` |
| `GET` | `/api/changes/{id}` | — | `200 ChangeEvaluation` / `404` | `PolicyEndpoint` ← `PolicyView` |
| `GET` | `/api/changes/{id}/gate` | — | `200 { changeId, gateStatus, reason }` | `PolicyEndpoint` ← `PolicyView` |
| `GET` | `/api/changes/sse` | — | `text/event-stream` | `PolicyEndpoint` ← `PolicyView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitChangeRequest (request body)

```json
{
  "changeTitle": "Enable public access on audit-log S3 bucket",
  "changeType": "terraform-plan",
  "rawPayload": "resource \"aws_s3_bucket\" \"audit-logs\" {\n  bucket = \"acme-audit-logs\"\n}\n\nresource \"aws_s3_bucket_public_access_block\" \"audit-logs\" {\n  bucket = aws_s3_bucket.audit-logs.id\n  block_public_acls   = false\n  block_public_policy = false\n}",
  "rules": [
    {
      "ruleId": "infra-s3-public-access-block",
      "text": "All S3 buckets must have block_public_acls, block_public_policy, ignore_public_acls, and restrict_public_buckets set to true.",
      "category": "network-access",
      "severityFloor": "CRITICAL"
    },
    {
      "ruleId": "infra-encryption-at-rest",
      "text": "All S3 buckets must declare a server_side_encryption_configuration block with AES256 or aws:kms.",
      "category": "encryption",
      "severityFloor": "HIGH"
    }
  ],
  "submittedBy": "platform-engineer-42"
}
```

### ChangeEvaluation (response body)

```json
{
  "changeId": "c-7a3...",
  "request": {
    "changeId": "c-7a3...",
    "changeTitle": "Enable public access on audit-log S3 bucket",
    "changeType": "terraform-plan",
    "rawPayload": "(raw payload preserved for audit)",
    "rules": [
      { "ruleId": "infra-s3-public-access-block", "text": "...", "category": "network-access", "severityFloor": "CRITICAL" }
    ],
    "submittedBy": "platform-engineer-42",
    "submittedAt": "2026-06-28T14:22:00Z"
  },
  "validated": {
    "changeType": "terraform-plan",
    "affectedResources": ["aws_s3_bucket.audit-logs", "aws_s3_bucket_public_access_block.audit-logs"],
    "normalizedPayload": "resource \"aws_s3_bucket\" \"audit-logs\" { ... }"
  },
  "decision": {
    "outcome": "DENY",
    "rationale": "S3 bucket has public access enabled, violating the access-block rule at critical severity.",
    "violations": [
      {
        "ruleId": "infra-s3-public-access-block",
        "severity": "CRITICAL",
        "affectedResource": "aws_s3_bucket_public_access_block.audit-logs",
        "evidence": "block_public_acls = false",
        "recommendation": "Set block_public_acls, block_public_policy, ignore_public_acls, and restrict_public_buckets to true."
      }
    ],
    "decidedAt": "2026-06-28T14:22:18Z"
  },
  "gate": {
    "status": "BLOCKED",
    "reason": "PolicyDecision.outcome is DENY; deployment is blocked.",
    "evaluatedAt": "2026-06-28T14:22:18Z"
  },
  "status": "GATED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### Gate status response

```json
{
  "changeId": "c-7a3...",
  "gateStatus": "BLOCKED",
  "reason": "PolicyDecision.outcome is DENY; deployment is blocked."
}
```

CI systems poll `GET /api/changes/{id}/gate` before proceeding. A `gateStatus` of `BLOCKED` must cause the pipeline step to fail. A `gateStatus` of `OPEN` means the change may proceed to the next pipeline stage.

### SSE event format

```
event: change-update
data: { "changeId": "c-7a3...", "status": "DECISION_RECORDED", "decision": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `VALIDATED`, `ENFORCING`, `DECISION_RECORDED`, `GATED`, `FAILED`). Clients reconcile by `changeId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body. The gate endpoint may additionally require a CI-system service token.
