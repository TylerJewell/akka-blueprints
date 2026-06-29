# API contract — sre-incident-responder

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/incidents` | `{ "description": String, "severity": String, "reportedBy"?: String }` | `202 { "incidentId": String }` | `IncidentEndpoint` → `AlertQueue` |
| `GET` | `/api/incidents` | — | `200 [ IncidentRow... ]` | `IncidentEndpoint` ← `IncidentView` |
| `GET` | `/api/incidents/{id}` | — | `200 Incident` / `404` | `IncidentEndpoint` ← `IncidentEntity` |
| `GET` | `/api/incidents/sse` | — | `text/event-stream` (one event per incident change) | `IncidentEndpoint` ← `IncidentView` |
| `POST` | `/api/incidents/{id}/approve` | `{ "approved": boolean, "decidedBy": String, "reason": String }` | `200 { "incidentId": String, "approved": boolean }` | `IncidentEndpoint` → `ApprovalEntity` |
| `GET` | `/api/incidents/{id}/approval` | — | `200 { "pending": boolean, "action"?: RemediationAction, "requestedAt"?: Instant, "decision"?: ApprovalDecision }` | `IncidentEndpoint` ← `ApprovalEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `IncidentEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `IncidentEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `IncidentEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Incident (full form, returned by `GET /api/incidents/{id}`)

```json
{
  "incidentId": "inc-9a3…",
  "description": "CPU spike on payment-service: p99 latency 4.2s, error rate 12%",
  "severity": "HIGH",
  "status": "AWAITING_APPROVAL",
  "investigationLedger": {
    "facts": ["payment-service CPU at 98%", "error rate 12%", "connection pool timeouts in logs"],
    "hypotheses": ["thread-pool exhaustion", "upstream dependency timeout"],
    "probePlan": [
      "Fetch CPU and latency metrics for payment-service",
      "Fetch error logs for payment-service",
      "Lookup service-restart runbook"
    ],
    "currentProbe": null
  },
  "remediationLedger": {
    "proposedActions": [
      {
        "actionKind": "RESTART_SERVICE",
        "target": "payment-service",
        "parameters": "rolling-restart",
        "rationale": "Connection pool exhaustion confirmed; rolling restart flushes stale connections",
        "estimatedImpact": "MEDIUM"
      }
    ],
    "lastDecision": null,
    "outcomes": []
  },
  "report": null,
  "evalScore": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:12:00Z",
  "finishedAt": null
}
```

### Incident (list form, returned by `GET /api/incidents`)

The list form is `IncidentRow` — same top-level fields as `Incident`, but `investigationLedger.probeEntries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count. The UI fetches the full incident by id on row expand.

### Approval state (returned by `GET /api/incidents/{id}/approval`)

```json
{
  "pending": true,
  "action": {
    "actionKind": "RESTART_SERVICE",
    "target": "payment-service",
    "parameters": "rolling-restart",
    "rationale": "Connection pool exhaustion confirmed; rolling restart flushes stale connections",
    "estimatedImpact": "MEDIUM"
  },
  "requestedAt": "2026-06-28T09:14:22Z",
  "decision": null
}
```

### SSE event format

```
event: incident-update
data: { "incidentId": "inc-9a3…", "status": "AWAITING_APPROVAL", ... }
```

One event per state transition. Clients reconcile by `incidentId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "cascade risk detected", "haltedAt": "2026-06-28T09:15:01Z" }
```

And `event: approval-update` when an approval state changes:

```
event: approval-update
data: { "incidentId": "inc-9a3…", "pending": false, "approved": true, "decidedBy": "sre-on-call" }
```
