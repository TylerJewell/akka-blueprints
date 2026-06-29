# API contract — durable-agent-baseline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/work-orders` | `SubmitWorkOrderRequest` | `201 { workOrderId }` | `WorkOrderEndpoint` → `WorkOrderEntity` + `WorkOrderWorkflow` |
| `GET` | `/api/work-orders` | — | `200 [ WorkOrderRow... ]` (newest-first) | `WorkOrderEndpoint` ← `WorkOrderView` |
| `GET` | `/api/work-orders/{id}` | — | `200 WorkOrderRow` / `404` | `WorkOrderEndpoint` ← `WorkOrderView` |
| `GET` | `/api/work-orders/sse` | — | `text/event-stream` | `WorkOrderEndpoint` ← `WorkOrderView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `WorkOrderEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `WorkOrderEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `WorkOrderEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitWorkOrderRequest (request body)

```json
{
  "title": "Nightly data pipeline — 2026-06-28",
  "steps": [
    {
      "stepId": "fetch-source-data",
      "description": "Retrieve the latest batch records from the upstream staging table.",
      "maxRetries": 2
    },
    {
      "stepId": "validate-schema",
      "description": "Check that every record conforms to the expected field schema.",
      "maxRetries": 1
    },
    {
      "stepId": "transform-and-load",
      "description": "Apply transformation rules and load into the target table.",
      "maxRetries": 2
    },
    {
      "stepId": "notify-downstream",
      "description": "Send completion notification to downstream consumers.",
      "maxRetries": 1
    }
  ],
  "submittedBy": "pipeline-scheduler-01"
}
```

### WorkOrderRow (response body)

```json
{
  "workOrderId": "wo-7a3...",
  "order": {
    "workOrderId": "wo-7a3...",
    "title": "Nightly data pipeline — 2026-06-28",
    "steps": [
      { "stepId": "fetch-source-data", "description": "...", "maxRetries": 2 }
    ],
    "submittedBy": "pipeline-scheduler-01",
    "submittedAt": "2026-06-28T01:00:00Z"
  },
  "result": {
    "resolution": "COMPLETED",
    "summary": "All four steps completed. Transform step required one retry due to a transient write timeout.",
    "outcomes": [
      {
        "stepId": "fetch-source-data",
        "status": "COMPLETED",
        "output": "Retrieved 4,820 records.",
        "retryCount": 0,
        "startedAt": "2026-06-28T01:00:02Z",
        "finishedAt": "2026-06-28T01:00:06Z"
      },
      {
        "stepId": "transform-and-load",
        "status": "COMPLETED",
        "output": "Loaded 4,820 records; 0 rejected.",
        "retryCount": 1,
        "startedAt": "2026-06-28T01:00:12Z",
        "finishedAt": "2026-06-28T01:00:31Z"
      }
    ],
    "decidedAt": "2026-06-28T01:00:42Z"
  },
  "eval": {
    "score": 3,
    "rationale": "Completed with one step retry; latency was within threshold; no stall alerts.",
    "evaluatedAt": "2026-06-28T01:00:43Z"
  },
  "stepOutcomes": [],
  "alerts": [],
  "status": "EVALUATED",
  "createdAt": "2026-06-28T01:00:00Z",
  "finishedAt": "2026-06-28T01:00:43Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: work-order-update
data: { "workOrderId": "wo-7a3...", "status": "STEP_COMPLETED", "stepOutcomes": [ ... ], ... }
```

One event per state transition (`INITIATED`, `RUNNING`, `STEP_COMPLETED`, `COMPLETED`, `EVALUATED`, `STALLED`, `FAILED`). Clients reconcile by `workOrderId`; an event always carries the full row at the moment of transition, so a late-joining client or one that reconnected after a JVM restart never needs to replay history.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
