# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/incidents` | `IncidentSignal` | `{ "incidentId": "uuid" }` | `IncidentEndpoint` → `IncidentQueue` |
| GET | `/api/incidents` | — | `{ "incidents": [IncidentRow, ...] }` | `IncidentEndpoint` → `IncidentView` |
| GET | `/api/incidents?status=AWAITING_APPROVAL` | — | filtered list (client-side filter) | `IncidentEndpoint` |
| GET | `/api/incidents/{id}` | — | `IncidentRow` or 404 | `IncidentEndpoint` |
| POST | `/api/incidents/{id}/approve` | `{ "officerId": "string", "reason": "string?" }` | `{ "status": "APPROVED" }` | `IncidentEndpoint` → `ApprovalEntity` |
| POST | `/api/incidents/{id}/reject` | `{ "officerId": "string", "reason": "string" }` | `{ "status": "REJECTED" }` | `IncidentEndpoint` → `ApprovalEntity` |
| GET | `/api/incidents/sse` | — | `text/event-stream` of `IncidentRow` | `IncidentEndpoint` → `IncidentView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `IncidentEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `IncidentEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `IncidentEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/incidents` request:

```json
{
  "assetId": "prod-api-gateway-01",
  "assetType": "api-gateway",
  "signalDescription": "Unusual spike in 401 responses from external IPs, 3400 req/min over baseline",
  "reportedBy": "soc-monitor-agent",
  "initialSeverity": "HIGH"
}
```

`IncidentRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "incidentId": "uuid",
  "assetId": "string",
  "assetType": "string",
  "signalDescription": "string",
  "reportedBy": "string",
  "status": "RECEIVED | TRIAGING | AWAITING_APPROVAL | MITIGATED | REJECTED | DEGRADED",
  "severity": "LOW | MEDIUM | HIGH | CRITICAL",
  "vulnerabilities": {
    "vulnerabilities": [
      { "cveId": "CVE-2024-12345", "description": "...", "cvssScore": 8.1, "affectedComponent": "...", "source": "NVD 2024" }
    ],
    "aggregateSeverity": "HIGH",
    "scannedAt": "ISO-8601"
  },
  "threatContext": {
    "actors": [
      { "actorGroup": "APT-28", "attackPattern": "Credential Stuffing — T1110.004", "targetProfile": "...", "historicalPrecedent": "...", "contextAt": "ISO-8601" }
    ],
    "summary": "string",
    "gatheredAt": "ISO-8601"
  },
  "triageReport": {
    "summary": "string",
    "riskLevel": "HIGH",
    "mitigationPlan": "string",
    "triageAt": "ISO-8601"
  },
  "approval": {
    "officerId": "string",
    "decision": "APPROVED | REJECTED",
    "reason": "string or null",
    "decidedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "receivedAt": "ISO-8601",
  "resolvedAt": "ISO-8601 or null"
}
```

`POST /api/incidents/{id}/approve` request:

```json
{ "officerId": "officer-jane-smith", "reason": "Validated CVE and attack pattern; mitigation plan is sound." }
```

`POST /api/incidents/{id}/reject` request:

```json
{ "officerId": "officer-jane-smith", "reason": "Asset is already isolated; mitigation plan is redundant." }
```

## SSE event format

`GET /api/incidents/sse` emits one event per incident change:

```
event: incident
data: { ...IncidentRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `incidentId`. No polling.
