# API contract — secops-triage

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/findings` | `IngestFindingRequest` | `201 { findingId }` | `FindingEndpoint` → `FindingEntity` |
| `GET` | `/api/findings` | — | `200 [ FindingRow... ]` (newest-first) | `FindingEndpoint` ← `FindingView` |
| `GET` | `/api/findings/{id}` | — | `200 FindingRow` / `404` | `FindingEndpoint` ← `FindingView` |
| `GET` | `/api/findings/sse` | — | `text/event-stream` | `FindingEndpoint` ← `FindingView` |
| `POST` | `/api/findings/{id}/approve` | `ApprovalRequest` | `204` | `FindingEndpoint` → `ApprovalEntity` |
| `POST` | `/api/findings/{id}/reject` | `ApprovalRequest` | `204` | `FindingEndpoint` → `ApprovalEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### IngestFindingRequest (request body)

```json
{
  "cveId": "CVE-2024-12345",
  "cvssScore": 9.8,
  "affectedAsset": "container-orchestrator-prod-01",
  "description": "A container escape vulnerability in the container runtime allows an attacker with container-level access to gain host-level code execution via a crafted seccomp profile bypass.",
  "submittedBy": "scanner-crowdstrike-01"
}
```

### ApprovalRequest (request body)

```json
{
  "analystId": "analyst-jsmith",
  "reason": "Patch verified in staging; approved for immediate production deployment."
}
```

### FindingRow (response body)

```json
{
  "findingId": "f-4a7...",
  "raw": {
    "cveId": "CVE-2024-12345",
    "cvssScore": 9.8,
    "affectedAsset": "container-orchestrator-prod-01",
    "description": "A container escape vulnerability ...",
    "submittedBy": "scanner-crowdstrike-01",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "enriched": {
    "raw": { "...": "same as above" },
    "assetContext": {
      "assetId": "container-orchestrator-prod-01",
      "assetCriticality": "TIER1",
      "internetFacing": true,
      "existingMitigations": "Network policy restricts egress; no host-pid sharing.",
      "ownerTeam": "platform-infra"
    },
    "threatIntelSummary": "Exploit code publicly available on GitHub since 2024-06-20. Active exploitation observed in honeypot networks.",
    "exploitInWild": true,
    "enrichedAt": "2026-06-28T14:00:01Z"
  },
  "verdict": {
    "priority": "CRITICAL_IMMEDIATE",
    "riskRationale": "CVE-2024-12345 scores 9.8 and affects a TIER1 container host directly reachable from the internet. A working public exploit is circulating in the wild, meaning the mean time to exploitation for unpatched hosts is measured in hours. No compensating controls are sufficient given the host's exposure.",
    "recommendedAction": "Apply the vendor patch immediately and isolate the affected host from the container network until the patch is confirmed deployed.",
    "requiresApproval": true,
    "decidedAt": "2026-06-28T14:00:28Z"
  },
  "approval": {
    "findingId": "f-4a7...",
    "status": "GRANTED",
    "analystId": "analyst-jsmith",
    "reason": "Patch verified in staging; approved for immediate production deployment.",
    "decidedAt": "2026-06-28T14:05:10Z"
  },
  "driftAlert": null,
  "status": "REMEDIATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:05:11Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: finding-update
data: { "findingId": "f-4a7...", "status": "PENDING_APPROVAL", "verdict": { ... }, ... }
```

One event per state transition (`INGESTED`, `ENRICHED`, `TRIAGING`, `VERDICT_RECORDED`, `PENDING_APPROVAL`, `REMEDIATED`, `REMEDIATION_REJECTED`, `MONITORED`, `ACCEPTED`, `FAILED`). Clients reconcile by `findingId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

A separate `drift-alert` event type is emitted when `DriftAlertRaised` lands on the sentinel entity:

```
event: drift-alert
data: { "alertId": "da-...", "description": "CRITICAL_IMMEDIATE findings dropped from 22% to 4% over the last 50 triages — baseline deviation exceeds threshold.", "baselineCriticalPct": 22.0, "observedCriticalPct": 4.0, "detectedAt": "2026-06-28T18:00:00Z" }
```

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware, source `submittedBy` from the authenticated scanner identity, and restrict `approve`/`reject` paths to authenticated analysts.
