# API contract — durable-workflow-backed-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/incidents` | `SubmitIncidentRequest` | `201 { incidentId }` | `RemediationEndpoint` → `IncidentEntity` |
| `GET` | `/api/incidents` | — | `200 [ IncidentRecord... ]` (newest-first) | `RemediationEndpoint` ← `IncidentView` |
| `GET` | `/api/incidents/{id}` | — | `200 IncidentRecord` / `404` | `RemediationEndpoint` ← `IncidentView` |
| `GET` | `/api/incidents/sse` | — | `text/event-stream` | `RemediationEndpoint` ← `IncidentView` |
| `GET` | `/api/durability/latest` | — | `200 DurabilityReport` / `204` | `RemediationEndpoint` ← `DurabilityReportEntity` |
| `POST` | `/api/durability/trigger` | — | `202 { reportId }` | `RemediationEndpoint` → `DurabilityEvaluator` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `RemediationEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `RemediationEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `RemediationEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitIncidentRequest (request body)

```json
{
  "alertId": "high-latency-api-gateway",
  "serviceId": "api-gateway"
}
```

### IncidentRecord (response body)

```json
{
  "incidentId": "inc-5af9c2...",
  "alertId": "high-latency-api-gateway",
  "serviceId": "api-gateway",
  "diagnosis": {
    "rootCause": "P99 latency spike to 4200ms caused by connection pool exhaustion under sustained load.",
    "affectedService": "api-gateway",
    "severity": "critical",
    "evidenceMetrics": [
      { "metricName": "latency_p99_ms", "value": 4200.0, "unit": "ms", "sampledAt": "2026-06-29T02:00:00Z" }
    ],
    "evidenceLogs": [
      { "level": "ERROR", "message": "Connection pool exhausted: timeout waiting for connection", "serviceId": "api-gateway", "timestamp": "2026-06-29T02:00:05Z" }
    ],
    "diagnosedAt": "2026-06-29T02:00:10Z"
  },
  "remediation": {
    "actionsApplied": [
      { "receiptId": "r-4a2b1c3d", "action": "increase_connection_pool_size", "target": "api-gateway", "issuedAt": "2026-06-29T02:00:15Z" }
    ],
    "outcome": "applied",
    "remediatedAt": "2026-06-29T02:00:20Z"
  },
  "verification": {
    "resolved": true,
    "healthChecks": [
      { "serviceId": "api-gateway", "healthy": true, "latencyP99Ms": 180.0, "checkedAt": "2026-06-29T02:00:35Z" }
    ],
    "resolutionCheck": {
      "incidentId": "inc-5af9c2...",
      "resolved": true,
      "evidence": "P99 latency returned to 180ms; no connection pool errors in the trailing 5-minute window.",
      "checkedAt": "2026-06-29T02:00:35Z"
    },
    "verifiedAt": "2026-06-29T02:00:35Z"
  },
  "status": "VERIFIED",
  "createdAt": "2026-06-29T02:00:00Z",
  "closedAt": "2026-06-29T02:00:35Z",
  "budgetViolations": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `budgetViolations` is an array — empty on the happy path, populated when the budget-cap guardrail fired during this incident:

```json
"budgetViolations": [
  {
    "phase": "DIAGNOSE",
    "tool": "fetchLogs",
    "callsUsed": 8,
    "cap": 8,
    "violatedAt": "2026-06-29T02:00:08Z"
  }
]
```

### DurabilityReport (response body for `/api/durability/latest`)

```json
{
  "reportId": "dr-9c0e44d1",
  "windowStart": "2026-06-29T01:00:00Z",
  "windowEnd": "2026-06-29T02:00:00Z",
  "incidentsEvaluated": 12,
  "completionRatePercent": 91.7,
  "budgetViolationRatePercent": 8.3,
  "mttrMinutes": 2.4,
  "timeoutRatePercent": 0.0,
  "generatedAt": "2026-06-29T02:00:05Z"
}
```

`204` is returned when no report has been generated yet (timer has not fired and no manual trigger has been issued).

### SSE event format

```
event: incident-update
data: { "incidentId": "inc-5af9c2...", "status": "DIAGNOSED", "diagnosis": { ... }, ... }
```

One event per state transition (`CREATED`, `DIAGNOSING`, `DIAGNOSED`, `REMEDIATING`, `REMEDIATED`, `VERIFYING`, `VERIFIED`, `FAILED`) and one per `BudgetExhausted` audit event:

```
event: incident-budget-exceeded
data: { "incidentId": "inc-5af9c2...", "phase": "DIAGNOSE", "tool": "fetchLogs", "callsUsed": 8, "cap": 8, "violatedAt": "..." }
```

Clients reconcile by `incidentId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitIncidentRequest` record and the `IncidentCreated` event to carry it.
