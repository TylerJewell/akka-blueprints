# API contract — wellness-check-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/campaigns` | `ScheduleCampaignRequest` | `201 { campaignId }` | `CheckInEndpoint` → `CampaignEntity` |
| `GET` | `/api/campaigns` | — | `200 [ Campaign... ]` (newest-first) | `CheckInEndpoint` ← `CampaignView` |
| `GET` | `/api/campaigns/{id}` | — | `200 Campaign` / `404` | `CheckInEndpoint` ← `CampaignView` |
| `POST` | `/api/check-ins` | `ReceiveCheckInRequest` | `201 { checkInId }` | `CheckInEndpoint` → `CheckInEntity` |
| `GET` | `/api/check-ins` | — | `200 [ CheckIn... ]` (newest-first) | `CheckInEndpoint` ← `CheckInView` |
| `GET` | `/api/check-ins/{id}` | — | `200 CheckIn` / `404` | `CheckInEndpoint` ← `CheckInView` |
| `GET` | `/api/check-ins/sse` | — | `text/event-stream` | `CheckInEndpoint` ← `CheckInView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### ScheduleCampaignRequest (request body)

```json
{
  "campaignName": "Q2 Morale Pulse — Engineering",
  "audienceLabel": "engineering-all",
  "questions": [
    {
      "questionId": "q-energy",
      "text": "How would you describe your energy level this week?",
      "kind": "OPEN_ENDED"
    },
    {
      "questionId": "q-support",
      "text": "Do you feel adequately supported by your team and manager?",
      "kind": "YES_NO"
    },
    {
      "questionId": "q-workload",
      "text": "On a scale of 1–5, how manageable is your current workload?",
      "kind": "LIKERT_1_5"
    }
  ],
  "scheduledAt": "2026-07-01T09:00:00Z",
  "createdBy": "hr-coordinator-07"
}
```

### ReceiveCheckInRequest (request body)

```json
{
  "campaignId": "c-a1b2...",
  "employeeRef": "emp-ref-4891",
  "answers": {
    "q-energy": "Feeling pretty drained honestly, the last sprint was rough.",
    "q-support": "no",
    "q-workload": "2"
  }
}
```

### Campaign (response body)

```json
{
  "campaignId": "c-a1b2...",
  "request": {
    "campaignId": "c-a1b2...",
    "campaignName": "Q2 Morale Pulse — Engineering",
    "audienceLabel": "engineering-all",
    "questions": [
      { "questionId": "q-energy", "text": "...", "kind": "OPEN_ENDED" }
    ],
    "scheduledAt": "2026-07-01T09:00:00Z",
    "createdBy": "hr-coordinator-07"
  },
  "status": "ACTIVE",
  "responsesReceived": 12,
  "highCount": 4,
  "moderateCount": 5,
  "lowCount": 2,
  "crisisCount": 1,
  "createdAt": "2026-06-28T14:00:00Z",
  "completedAt": null
}
```

### CheckIn (response body)

```json
{
  "checkInId": "ci-9f3...",
  "campaignId": "c-a1b2...",
  "response": {
    "checkInId": "ci-9f3...",
    "campaignId": "c-a1b2...",
    "employeeRef": "emp-ref-4891",
    "answers": {
      "q-energy": "Feeling pretty drained honestly, the last sprint was rough.",
      "q-support": "no",
      "q-workload": "2"
    },
    "receivedAt": "2026-06-28T14:05:00Z"
  },
  "sanitized": {
    "redactedAnswers": {
      "q-energy": "Feeling pretty drained honestly, the last sprint was rough.",
      "q-support": "no",
      "q-workload": "2"
    },
    "specialCategoriesFound": []
  },
  "analysis": {
    "moraleLevel": "LOW",
    "interpretation": "The employee reports low energy and describes feeling unsupported. Workload is rated 2/5.",
    "crisisFlag": false,
    "recommendation": "Schedule a 1:1 with the employee's manager within 5 business days.",
    "analysedAt": "2026-06-28T14:05:22Z"
  },
  "surveillance": {
    "riskFlagRaised": false,
    "rationale": "Campaign LOW+CRISIS ratio is 0.25; below threshold.",
    "evaluatedAt": "2026-06-28T14:05:23Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:05:00Z",
  "finishedAt": "2026-06-28T14:05:23Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: check-in-update
data: { "checkInId": "ci-9f3...", "status": "ANALYSIS_RECORDED", "analysis": { ... }, ... }
```

One event per state transition (`RECEIVED`, `SANITIZED`, `ANALYSING`, `ANALYSIS_RECORDED`, `ESCALATED`, `EVALUATED`, `FAILED`). Clients reconcile by `checkInId`; an event always carries the full row at the moment of transition so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware. `employeeRef` should be a pseudonymised token supplied by the identity layer, not a resolvable employee name or email — this is the pseudonymisation obligation that accompanies special-category data processing.
