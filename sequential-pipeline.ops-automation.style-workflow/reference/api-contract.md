# API contract — workflow-orchestration

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `WorkflowRunEndpoint` → `WorkflowRunEntity` |
| `GET` | `/api/runs` | — | `200 [ WorkflowRunRecord... ]` (newest-first) | `WorkflowRunEndpoint` ← `WorkflowRunView` |
| `GET` | `/api/runs/{id}` | — | `200 WorkflowRunRecord` / `404` | `WorkflowRunEndpoint` ← `WorkflowRunView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `WorkflowRunEndpoint` ← `WorkflowRunView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "workflowId": "database-migration-v3"
}
```

### WorkflowRunRecord (response body)

```json
{
  "runId": "r-8bc3e1...",
  "workflowId": "database-migration-v3",
  "validation": {
    "workflowId": "database-migration-v3",
    "validations": [
      { "stepId": "backup-db", "status": "OK", "warning": "" },
      { "stepId": "apply-schema", "status": "OK", "warning": "" },
      { "stepId": "run-migrations", "status": "WARN", "warning": "Migration 0042 has no rollback defined" }
    ],
    "allPassed": true,
    "validatedAt": "2026-06-28T10:00:00Z"
  },
  "execution": {
    "workflowId": "database-migration-v3",
    "outcomes": [
      { "stepId": "backup-db", "status": "SUCCEEDED", "durationMs": 340, "detail": "Backup written to /var/backup/db-20260628.gz" },
      { "stepId": "apply-schema", "status": "SUCCEEDED", "durationMs": 120, "detail": "" },
      { "stepId": "run-migrations", "status": "SUCCEEDED", "durationMs": 890, "detail": "3 migrations applied" }
    ],
    "executedAt": "2026-06-28T10:00:12Z"
  },
  "result": {
    "runId": "r-8bc3e1...",
    "workflowId": "database-migration-v3",
    "summary": {
      "workflowId": "database-migration-v3",
      "totalSteps": 3,
      "succeeded": 3,
      "failed": 0,
      "skipped": 0
    },
    "stepOutcomes": [
      { "stepId": "backup-db", "status": "SUCCEEDED", "durationMs": 340, "detail": "Backup written to /var/backup/db-20260628.gz" },
      { "stepId": "apply-schema", "status": "SUCCEEDED", "durationMs": 120, "detail": "" },
      { "stepId": "run-migrations", "status": "SUCCEEDED", "durationMs": 890, "detail": "3 migrations applied" }
    ],
    "notification": {
      "channel": "ops-alerts",
      "messageId": "msg-f9a44c...",
      "sentAt": "2026-06-28T10:00:14Z"
    },
    "completedAt": "2026-06-28T10:00:14Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Step coverage, phantom-step check, valid-outcome check, and count parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:14Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:14Z",
  "guardrailRejections": [
    {
      "stage": "VALIDATE",
      "tool": "dispatchNotification",
      "reason": "stage-violation: dispatchNotification requires status in {EXECUTED, NOTIFYING} with execution present, saw VALIDATING",
      "rejectedAt": "2026-06-28T10:00:01Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

### SSE event format

```
event: run-update
data: { "runId": "r-8bc3e1...", "status": "EXECUTED", "execution": { ... }, ... }
```

One event per state transition (`CREATED`, `VALIDATING`, `VALIDATED`, `EXECUTING`, `EXECUTED`, `NOTIFYING`, `NOTIFIED`, `EVALUATED`, `FAILED`) and one per `StageGuardrailRejected` audit event:

```
event: run-rejection
data: { "runId": "r-8bc3e1...", "stage": "VALIDATE", "tool": "dispatchNotification", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitRunRequest` record and the `RunCreated` event to carry it.
