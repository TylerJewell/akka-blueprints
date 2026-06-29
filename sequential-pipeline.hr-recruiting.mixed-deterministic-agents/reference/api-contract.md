# API contract — score-aggregator

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/applications` | `SubmitApplicationRequest` | `201 { applicationId }` | `ApplicationEndpoint` → `ApplicationEntity` |
| `GET` | `/api/applications` | — | `200 [ ApplicationRecord... ]` (newest-first) | `ApplicationEndpoint` ← `ApplicationView` |
| `GET` | `/api/applications/{id}` | — | `200 ApplicationRecord` / `404` | `ApplicationEndpoint` ← `ApplicationView` |
| `GET` | `/api/applications/sse` | — | `text/event-stream` | `ApplicationEndpoint` ← `ApplicationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitApplicationRequest (request body)

```json
{
  "candidateName": "Alice Chen",
  "resumeText": "Staff engineer with 10 years in distributed systems...",
  "targetRole": "staff-software-engineer"
}
```

### ApplicationRecord (response body)

```json
{
  "applicationId": "app-3bf7e1...",
  "candidateName": "Alice Chen",
  "resumeText": "Staff engineer with 10 years in distributed systems...",
  "targetRole": "staff-software-engineer",
  "screenResult": {
    "candidateId": "alice-chen",
    "role": "staff-software-engineer",
    "passedInitialScreen": true,
    "skillMatches": [
      { "skill": "Java", "matched": true, "evidence": "Led Java backend migration for payments platform." },
      { "skill": "Distributed systems", "matched": true, "evidence": "Designed event-sourced order service handling 10k req/s." }
    ],
    "fitSummary": "Strong match on core engineering and leadership criteria.",
    "screenedAt": "2026-06-28T10:00:05Z"
  },
  "candidateScore": {
    "totalScore": 87,
    "dimensionScores": [
      { "dimension": "skills-match", "score": 27, "rationale": "3 of 3 required skills matched." },
      { "dimension": "experience-years", "score": 22, "rationale": "10 years meets 8-year floor." },
      { "dimension": "role-seniority", "score": 18, "rationale": "Prior staff title matches target level." },
      { "dimension": "location-preference", "score": 10, "rationale": "Remote preferred; role allows remote." },
      { "dimension": "screen-flag", "score": 10, "rationale": "Passed initial screen." }
    ],
    "scoredAt": "2026-06-28T10:00:06Z"
  },
  "recommendation": {
    "decision": "ADVANCE",
    "rationale": "CandidateScore of 87/100 with all dimensions at or above threshold.",
    "recommendedAt": "2026-06-28T10:00:20Z"
  },
  "aggregatedScore": {
    "score": {
      "totalScore": 87,
      "dimensionScores": [...],
      "scoredAt": "2026-06-28T10:00:06Z"
    },
    "recommendation": {
      "decision": "ADVANCE",
      "rationale": "CandidateScore of 87/100 with all dimensions at or above threshold.",
      "recommendedAt": "2026-06-28T10:00:20Z"
    }
  },
  "statusUpdate": {
    "applicationId": "app-3bf7e1...",
    "decision": "ADVANCE",
    "score": 87,
    "notificationPayload": "{\"applicationId\":\"app-3bf7e1...\",\"decision\":\"ADVANCE\",\"score\":87,\"notifiedAt\":\"2026-06-28T10:00:21Z\"}",
    "notifiedAt": "2026-06-28T10:00:21Z"
  },
  "gateResult": {
    "pass": true,
    "reason": "all checks passed",
    "evaluatedAt": "2026-06-28T10:00:21Z"
  },
  "status": "NOTIFIED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:21Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: application-update
data: { "applicationId": "app-3bf7e1...", "status": "SCORED", "candidateScore": { ... }, ... }
```

One event per state transition (`CREATED`, `SCREENING`, `SCREENED`, `SCORING`, `SCORED`, `RECOMMENDING`, `RECOMMENDED`, `NOTIFIED`, `FAILED`) and one additional event when the gate result is recorded:

```
event: application-gate-result
data: { "applicationId": "app-3bf7e1...", "pass": false, "reason": "DimensionScore count was 3; expected 5.", "evaluatedAt": "..." }
```

Clients reconcile by `applicationId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitApplicationRequest` record and the `ApplicationCreated` event to carry it. In a production HR deployment, access to resume text should be restricted to authorised recruiters and hiring managers.
