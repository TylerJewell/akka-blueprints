# API contract — siem-enrichment

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/alerts` | `SubmitAlertRequest` | `201 { alertId }` | `AlertEndpoint` → `AlertEntity` |
| `GET` | `/api/alerts` | — | `200 [ AlertRecord... ]` (newest-first) | `AlertEndpoint` ← `AlertView` |
| `GET` | `/api/alerts/{id}` | — | `200 AlertRecord` / `404` | `AlertEndpoint` ← `AlertView` |
| `GET` | `/api/alerts/sse` | — | `text/event-stream` | `AlertEndpoint` ← `AlertView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAlertRequest (request body)

```json
{
  "rawAlertJson": "{\"ruleName\":\"Suspicious SMB lateral movement detected\",\"severity\":\"HIGH\",\"sourceIp\":\"10.0.1.45\",\"destinationIp\":\"10.0.2.12\",\"hostname\":\"workstation-045\"}"
}
```

### AlertRecord (response body)

```json
{
  "alertId": "a-3c8f1d...",
  "rawAlertJson": "{...}",
  "alertDetail": {
    "alertId": "a-3c8f1d...",
    "ruleName": "Suspicious SMB lateral movement detected",
    "severity": "HIGH",
    "sourceIp": "10.0.1.45",
    "destinationIp": "10.0.2.12",
    "hostname": "workstation-045",
    "rawFields": [
      { "key": "event.action", "value": "network-connection" },
      { "key": "process.name", "value": "cmd.exe" },
      { "key": "destination.port", "value": "445" }
    ],
    "detectedAt": "2026-06-28T09:14:22Z"
  },
  "enrichedAlert": {
    "alertDetail": { "...": "as above" },
    "attackPatterns": [
      {
        "patternId": "ap-3f8a22b1",
        "techniqueId": "T1021.002",
        "techniqueName": "Remote Services: SMB/Windows Admin Shares",
        "tactic": "Lateral Movement",
        "confidence": "HIGH",
        "indicator": "destination.port=445"
      }
    ],
    "derivedSeverity": "HIGH",
    "enrichedAt": "2026-06-28T09:14:35Z"
  },
  "triageTicket": {
    "alertId": "a-3c8f1d...",
    "title": "HIGH: Suspicious SMB lateral movement — T1021.002",
    "severity": "HIGH",
    "assignedTeam": "incident-response",
    "attackPatterns": [ { "...": "as above" } ],
    "zendeskRef": {
      "ticketId": "ZD-10042",
      "ticketUrl": "https://example.zendesk.com/agent/tickets/10042"
    },
    "summary": "Detected SMB lateral movement from 10.0.1.45 to 10.0.2.12 via cmd.exe. One ATT&CK technique mapped: T1021.002 (Lateral Movement, HIGH confidence). Assigned to incident-response team.",
    "createdAt": "2026-06-28T09:14:48Z"
  },
  "eval": {
    "score": 5,
    "rationale": "ATT&CK technique presence, technique validity, severity accuracy, and routing completeness all satisfied.",
    "evaluatedAt": "2026-06-28T09:14:49Z"
  },
  "status": "EVALUATED",
  "receivedAt": "2026-06-28T09:14:20Z",
  "finishedAt": "2026-06-28T09:14:49Z",
  "guardrailRejections": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `guardrailRejections` is an array — empty on the happy path, populated when the agent attempted a misordered tool call and the guardrail caught it.

### GuardrailRejection in the response (populated only on the violation path)

```json
{
  "guardrailRejections": [
    {
      "phase": "ENRICH",
      "tool": "openZendeskTicket",
      "reason": "scope-violation: openZendeskTicket requires status in {ENRICHED, TRIAGING} with enrichedAlert present, saw ENRICHING",
      "rejectedAt": "2026-06-28T09:14:33Z"
    }
  ]
}
```

### SSE event format

```
event: alert-update
data: { "alertId": "a-3c8f1d...", "status": "ENRICHED", "enrichedAlert": { ... }, ... }
```

One event per state transition (`RECEIVED`, `FETCHING`, `FETCHED`, `ENRICHING`, `ENRICHED`, `TRIAGING`, `TRIAGED`, `EVALUATED`, `FAILED`) and one per `GuardrailRejected` audit event:

```
event: alert-rejection
data: { "alertId": "a-3c8f1d...", "phase": "ENRICH", "tool": "openZendeskTicket", "reason": "...", "rejectedAt": "..." }
```

Clients reconcile by `alertId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitAlertRequest` record and the `AlertReceived` event to carry it.
