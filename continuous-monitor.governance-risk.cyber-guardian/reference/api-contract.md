# API contract — cyber-guardian-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/threats` | — | `200 [ IncidentRow... ]` (sorted newest-first) | `ThreatEndpoint` ← `IncidentView` |
| `GET` | `/api/threats/{id}` | — | `200 IncidentState` / `404` | `ThreatEndpoint` ← `IncidentView` |
| `POST` | `/api/threats/{id}/lift-halt` | `{ "liftedBy": String, "reason": String }` | `200 IncidentState` (status now REMEDIATION) | `ThreatEndpoint` → `IncidentEntity` |
| `GET` | `/api/threats/sse` | — | `text/event-stream` | `ThreatEndpoint` ← `IncidentView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/signal-categories` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncidentState (full detail)

```json
{
  "signalId": "sig-a3f…",
  "signal": {
    "signalId": "sig-a3f…",
    "sourceHost": "prod-db-01.internal",
    "category": "ransomware-detected",
    "rawPayload": "vssadmin delete shadows /all; .crypt extensions observed",
    "receivedAt": "2026-06-28T09:12:00Z"
  },
  "assessment": {
    "severity": "CRITICAL",
    "category": "Impact",
    "reasoning": "Ransomware execution confirmed via shadow-copy deletion and .crypt extension.",
    "confidenceScore": 0.97
  },
  "halt": {
    "signalId": "sig-a3f…",
    "isolatedHost": "prod-db-01.internal",
    "justification": "Ransomware execution confirmed: shadow copies deleted and .crypt extension observed.",
    "issuedAt": "2026-06-28T09:12:04Z"
  },
  "liftDecision": null,
  "playbook": {
    "steps": [
      "Verify host isolation at the network layer.",
      "Preserve a forensic image before any remediation.",
      "Identify the patient-zero process that spawned the ransomware.",
      "Rotate all credentials that were accessible from prod-db-01.internal.",
      "Restore from the last verified clean snapshot.",
      "Perform a full malware scan on adjacent hosts in the same subnet.",
      "Document the incident timeline in the ticketing system before closing."
    ],
    "estimatedEffort": "4-8h",
    "generatedAt": "2026-06-28T09:12:09Z"
  },
  "evalEvent": null,
  "status": "HALTED",
  "createdAt": "2026-06-28T09:12:00Z",
  "closedAt": null
}
```

### IncidentRow (list view — rawPayload omitted)

```json
{
  "signalId": "sig-a3f…",
  "sourceHost": "prod-db-01.internal",
  "category": "ransomware-detected",
  "severity": "CRITICAL",
  "status": "HALTED",
  "haltIssued": true,
  "playbookGenerated": true,
  "evalEventEmitted": false,
  "createdAt": "2026-06-28T09:12:00Z",
  "closedAt": null
}
```

### SSE event format

```
event: incident-update
data: { "signalId": "sig-a3f…", "status": "HALTED", "severity": "CRITICAL", ... }
```

One event per state transition. Clients reconcile by `signalId`.
