# API contract — nurse-handover

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/handovers` | `SubmitHandoverRequest` | `201 { handoverId }` | `HandoverEndpoint` → `HandoverEntity` |
| `GET` | `/api/handovers` | — | `200 [ Handover... ]` (newest-first) | `HandoverEndpoint` ← `HandoverView` |
| `GET` | `/api/handovers/{id}` | — | `200 Handover` / `404` | `HandoverEndpoint` ← `HandoverView` |
| `PATCH` | `/api/handovers/{id}/signoff` | `{ signedOffBy: String }` | `200` / `409` if already signed off | `HandoverEndpoint` → `HandoverEntity` |
| `GET` | `/api/handovers/sse` | — | `text/event-stream` | `HandoverEndpoint` ← `HandoverView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitHandoverRequest (request body)

```json
{
  "ward": "General Medicine",
  "shiftDate": "2026-06-28",
  "submittedBy": "Nurse Jordan Lee",
  "rawReport": "End of shift — 3 patients. Patient-A is post-appendectomy day 2 ...",
  "checklist": [
    {
      "itemId": "gm-vitals-stable",
      "description": "Confirm vital signs are documented and within normal range for each patient.",
      "urgency": "URGENT"
    },
    {
      "itemId": "gm-medications-reconciled",
      "description": "Verify discharge medications are reconciled and patient has been counselled.",
      "urgency": "ROUTINE"
    },
    {
      "itemId": "gm-pending-results",
      "description": "List any outstanding lab or imaging results and expected turnaround.",
      "urgency": "URGENT"
    }
  ]
}
```

### Handover (response body)

```json
{
  "handoverId": "h-3ac...",
  "request": {
    "handoverId": "h-3ac...",
    "ward": "General Medicine",
    "shiftDate": "2026-06-28",
    "submittedBy": "Nurse Jordan Lee",
    "rawReport": "(raw text preserved for audit)",
    "checklist": [
      { "itemId": "gm-vitals-stable", "description": "...", "urgency": "URGENT" }
    ],
    "submittedAt": "2026-06-28T07:00:00Z"
  },
  "sanitized": {
    "redactedReport": "End of shift — 3 patients. Patient-A is post-appendectomy day 2, MRN [REDACTED-MRN] ...",
    "phiCategoriesFound": ["mrn", "dob", "phone"]
  },
  "summary": {
    "patients": [
      {
        "patientRef": "Patient-A",
        "currentCondition": "Stable post-appendectomy, afebrile, tolerating oral intake.",
        "outstandingTasks": "gm-medications-reconciled: discharge medications not yet reconciled",
        "riskLevel": "MODERATE"
      }
    ],
    "outstandingTasks": [
      "Patient-B SpO2 monitoring every 2 hours (URGENT)",
      "Patient-A discharge medications reconciliation (ROUTINE)"
    ],
    "riskFlags": [
      "Patient-B SpO2 below target threshold — escalate if drops below 92%"
    ],
    "narrativeSummary": "General Medicine ward has 3 patients at end of shift. Patient-B requires close respiratory monitoring and is at HIGH risk. Two patients have outstanding medication tasks. No immediate critical escalations.",
    "summarizedAt": "2026-06-28T07:00:18Z"
  },
  "signoff": {
    "signedOffBy": "Dr. M. Patel",
    "signedOffAt": "2026-06-28T07:12:44Z"
  },
  "status": "SIGNED_OFF",
  "createdAt": "2026-06-28T07:00:00Z",
  "finishedAt": "2026-06-28T07:12:44Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: handover-update
data: { "handoverId": "h-3ac...", "status": "AWAITING_SIGNOFF", "summary": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `SUMMARIZING`, `SUMMARY_READY`, `AWAITING_SIGNOFF`, `SIGNED_OFF`, `FAILED`). Clients reconcile by `handoverId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and derive `submittedBy` / `signedOffBy` from the authenticated principal rather than the request body. The signoff endpoint in particular must verify that the caller is a credentialed clinician distinct from the shift-end submitter.
