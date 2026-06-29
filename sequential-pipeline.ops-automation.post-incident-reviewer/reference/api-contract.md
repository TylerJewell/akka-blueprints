# API contract — post-incident-reviewer

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/pir` | `SubmitPIRRequest` | `201 { pirId }` | `PIREndpoint` → `PIREntity` |
| `GET` | `/api/pir` | — | `200 [ PIRRecord... ]` (newest-first) | `PIREndpoint` ← `PIRView` |
| `GET` | `/api/pir/{id}` | — | `200 PIRRecord` / `404` | `PIREndpoint` ← `PIRView` |
| `POST` | `/api/pir/{id}/signoff` | `SignoffRequest` | `200 { pirId, status }` | `PIREndpoint` → `PIRWorkflow` |
| `GET` | `/api/pir/sse` | — | `text/event-stream` | `PIREndpoint` ← `PIRView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitPIRRequest (request body)

```json
{
  "incidentId": "INC-2026-0741"
}
```

### SignoffRequest (request body)

```json
{
  "approved": true,
  "comments": "Looks good — action items are well-scoped."
}
```

`comments` is optional. When `approved` is `false`, providing a `comments` string is strongly recommended; the rejection comment is stored on the entity and displayed in the UI.

### PIRRecord (response body)

```json
{
  "pirId": "pir-abc123",
  "incidentId": "INC-2026-0741",
  "evidenceLog": {
    "incident": {
      "incidentId": "INC-2026-0741",
      "title": "Database connection pool exhaustion — production API tier",
      "severity": "P2",
      "reportedBy": "on-call-sre@example.org",
      "detectedAt": "2026-06-10T02:14:00Z",
      "resolvedAt": "2026-06-10T03:47:00Z"
    },
    "timeline": [
      { "eventId": "evt-001", "occurredAt": "2026-06-10T02:14:00Z", "actor": "PagerDuty", "description": "Alert fired: DB connection pool > 95% utilization." },
      { "eventId": "evt-003", "occurredAt": "2026-06-10T02:41:00Z", "actor": "alice@example.org", "description": "Identified long-running query from batch job holding 120 connections." }
    ],
    "gatheredAt": "2026-06-28T10:00:00Z"
  },
  "impactAssessment": {
    "classification": { "severity": "P2", "affectedSystems": "production API tier", "usersAffected": 4200, "outageWindow": "PT1H33M" },
    "rootCause": {
      "rootCauseId": "rc-b9f2",
      "summary": "Batch job executed an unoptimized query that held 120 database connections for 27 minutes.",
      "contributingFactors": ["No connection timeout on batch job queries", "No alerting threshold below 95%"]
    },
    "assessedAt": "2026-06-28T10:00:10Z"
  },
  "review": {
    "pirId": "pir-abc123",
    "executiveSummary": "A P2 database connection pool exhaustion on 10 June 2026 caused a 93-minute production API degradation. The root cause was a batch job holding 120 connections. Three action items have been assigned.",
    "impactClassification": { "severity": "P2", "affectedSystems": "production API tier", "usersAffected": 4200, "outageWindow": "PT1H33M" },
    "timeline": [
      { "eventId": "evt-001", "occurredAt": "2026-06-10T02:14:00Z", "actor": "PagerDuty", "description": "Alert fired: DB connection pool > 95% utilization." },
      { "eventId": "evt-003", "occurredAt": "2026-06-10T02:41:00Z", "actor": "alice@example.org", "description": "Identified long-running query." }
    ],
    "rootCause": {
      "rootCauseId": "rc-b9f2",
      "summary": "Batch job executed an unoptimized query that held 120 database connections for 27 minutes.",
      "contributingFactors": ["No connection timeout on batch job queries", "No alerting threshold below 95%"]
    },
    "actionItems": [
      { "actionId": "ai-01", "description": "Add 30s query timeout to batch job DB sessions", "owner": "bob@example.org", "dueDate": "2026-07-05", "priority": "HIGH" }
    ],
    "draftedAt": "2026-06-28T10:00:20Z"
  },
  "signoffDecision": {
    "approved": true,
    "comments": "Looks good — action items are well-scoped.",
    "decidedBy": "alice@example.org",
    "decidedAt": "2026-06-28T10:05:00Z"
  },
  "status": "COMPLETE",
  "createdAt": "2026-06-28T09:59:50Z",
  "completedAt": "2026-06-28T10:05:00Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not yet been populated is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the DRAFT task response failed a quality check and the agent retried.

### SSE event format

```
event: pir-update
data: { "pirId": "pir-abc123", "status": "ASSESSED", "impactAssessment": { ... }, ... }
```

One event per state transition (`CREATED`, `GATHERING`, `GATHERED`, `ASSESSING`, `ASSESSED`, `DRAFTING`, `DRAFTED`, `AWAITING_SIGNOFF`, `COMPLETE`, `REJECTED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: pir-rejection
data: { "pirId": "pir-abc123", "check": "action-item-owner", "reason": "draft-quality-violation: actionItems[0].owner is null", "rejectedAt": "..." }
```

Clients reconcile by `pirId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal on `PIRCreated`, and a `decidedBy` field from the authenticated principal on `SignoffRecorded`. Extend `SubmitPIRRequest`, `SignoffRequest`, and the corresponding events to carry these fields.
