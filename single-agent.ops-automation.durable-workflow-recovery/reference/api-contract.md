# API contract — durable-workflow-recovery

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/executions` | `RegisterExecutionRequest` | `201 { executionId }` | `RecoveryEndpoint` → `ExecutionEntity` |
| `GET` | `/api/executions` | — | `200 [ Execution... ]` (newest-first) | `RecoveryEndpoint` ← `ExecutionView` |
| `GET` | `/api/executions/{id}` | — | `200 Execution` / `404` | `RecoveryEndpoint` ← `ExecutionView` |
| `GET` | `/api/executions/sse` | — | `text/event-stream` | `RecoveryEndpoint` ← `ExecutionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### RegisterExecutionRequest (request body)

```json
{
  "workflowId": "wf-etl-daily-ingestion-20260628",
  "workflowType": "etl-pipeline",
  "ownerTeam": "data-platform"
}
```

### Execution (response body)

```json
{
  "executionId": "ex-4a7...",
  "registration": {
    "executionId": "ex-4a7...",
    "workflowId": "wf-etl-daily-ingestion-20260628",
    "workflowType": "etl-pipeline",
    "ownerTeam": "data-platform",
    "registeredAt": "2026-06-28T09:00:00Z"
  },
  "snapshot": {
    "checkpoints": [
      {
        "checkpointId": "cp-extract-001",
        "phase": "extract",
        "elapsedMs": 4230,
        "outcome": "COMPLETED",
        "recordedAt": "2026-06-28T09:00:04Z"
      },
      {
        "checkpointId": "cp-transform-002",
        "phase": "transform",
        "elapsedMs": 8910,
        "outcome": "COMPLETED",
        "recordedAt": "2026-06-28T09:00:13Z"
      },
      {
        "checkpointId": "cp-load-003",
        "phase": "load",
        "elapsedMs": 0,
        "outcome": "PENDING",
        "recordedAt": "2026-06-28T09:00:13Z"
      }
    ],
    "totalExpected": 8,
    "completedCount": 2,
    "failedCount": 0,
    "stalledFor": "PT2M30S"
  },
  "decision": {
    "verdict": "RESUME",
    "rationale": "Extract and transform phases completed durably. Load is pending after a 2m30s stall consistent with a transient network interruption. Resuming from cp-load-003 is safe.",
    "checkpointStatuses": [
      {
        "checkpointId": "cp-extract-001",
        "phase": "extract",
        "outcome": "COMPLETED",
        "recommendedAction": "No action — checkpoint is durable."
      },
      {
        "checkpointId": "cp-transform-002",
        "phase": "transform",
        "outcome": "COMPLETED",
        "recommendedAction": "No action — checkpoint is durable."
      },
      {
        "checkpointId": "cp-load-003",
        "phase": "load",
        "outcome": "PENDING",
        "recommendedAction": "Retry from this checkpoint — prior state is durable."
      }
    ],
    "decidedAt": "2026-06-28T09:02:45Z"
  },
  "health": {
    "score": 4,
    "diagnosis": "Latency within baseline; retry saturation low; single pending checkpoint.",
    "evaluatedAt": "2026-06-28T09:02:46Z"
  },
  "status": "HEALTH_SCORED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:02:46Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: execution-update
data: { "executionId": "ex-4a7...", "status": "DECISION_RECORDED", "decision": { ... }, ... }
```

One event per state transition (`REGISTERED`, `RUNNING`, `STALLED`, `ANALYZING`, `DECISION_RECORDED`, `HEALTH_SCORED`, `ABORTED`, `FAILED`). Clients reconcile by `executionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `ownerTeam` from the authenticated principal's team claim rather than the request body.
