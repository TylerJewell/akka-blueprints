# API contract — incident-management

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/incidents` | — | `200 [ Incident... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `IncidentEndpoint` ← `IncidentView` |
| `GET` | `/api/incidents/{id}` | — | `200 Incident` / `404` | `IncidentEndpoint` ← `IncidentView` |
| `POST` | `/api/incidents` | `{ "reportedBy": String, "alertSource": String, "title": String, "description": String }` | `201 { "incidentId": String }` | `IncidentEndpoint` → `IncidentQueue` |
| `POST` | `/api/incidents/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Incident` (status now `RESOLVED`) / `409` if not `BLOCKED` | `IncidentEndpoint` → `IncidentEntity` |
| `GET` | `/api/incidents/sse` | — | `text/event-stream` | `IncidentEndpoint` ← `IncidentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundReport (request body for `POST /api/incidents`, minus `incidentId` and `reportedAt`)

```json
{
  "reportedBy": "sre-oncall",
  "alertSource": "datadog",
  "title": "Disk usage at 98% on prod-db-03",
  "description": "The /data volume on prod-db-03 is at 98% utilisation. Write errors appearing in application logs."
}
```

### Incident

```json
{
  "incidentId": "inc-7f4a2c91",
  "report": {
    "incidentId": "inc-7f4a2c91",
    "reportedBy": "sre-oncall",
    "alertSource": "datadog",
    "title": "Disk usage at 98% on prod-db-03",
    "description": "The /data volume on prod-db-03 is at 98%...",
    "reportedAt": "2026-06-28T09:10:00Z"
  },
  "enriched": {
    "incidentId": "inc-7f4a2c91",
    "title": "Disk usage at 98% on prod-db-03",
    "description": "The /data volume on prod-db-03 is at 98%...",
    "hostGroup": "prod-us-east-1",
    "serviceTier": "standard",
    "tags": ["disk", "storage"]
  },
  "classification": {
    "category": "INFRASTRUCTURE",
    "confidence": "high",
    "reason": "Host-level disk saturation is the primary signal."
  },
  "remediation": {
    "summary": "Disk saturation on prod-db-03 resolved by rotating the write-heavy log directory and restarting the storage agent.",
    "details": "1. SSH to prod-db-03. 2. Run logrotate -f /etc/logrotate.d/app. 3. Restart storage-agent service.",
    "action": "RESTART_SERVICE",
    "specialistTag": "infra",
    "proposedAt": "2026-06-28T09:10:14Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Category right; reason correctly identifies the disk saturation signal.",
    "scoredAt": "2026-06-28T09:10:11Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:17Z"
}
```

A blocked incident has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated incident has `classification.category = "AMBIGUOUS"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### ClassificationDecision (returned by `ClassifierAgent`, embedded in `Incident.classification`)

```json
{ "category": "INFRASTRUCTURE" | "APPLICATION" | "AMBIGUOUS",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### RemediationPlan (returned by either specialist, embedded in `Incident.remediation`)

```json
{ "summary": "Two-sentence max.",
  "details": "Step-by-step for the on-call engineer.",
  "action": "RESTART_SERVICE" | "SCALE_DEPLOYMENT" | "ROLLBACK_DEPLOY" | "OPEN_CHANGE_RECORD" | "PAGE_ON_CALL" | "INFO_PROVIDED",
  "specialistTag": "infra" | "app",
  "proposedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ToolCallGuardrail`, embedded in `Incident.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["peak-hours-restart", "critical-host-no-authority", "alert-token-echo", …],
  "rubricVersion": "v1" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Incident.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: incident-update
data: { "incidentId": "inc-…", "status": "CLASSIFIED", … full Incident JSON … }
```

One event per state transition on the `IncidentView`. Clients reconcile by `incidentId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/incidents`).
- `404` — unknown `incidentId`.
- `409` — `unblock` requested on an incident not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
