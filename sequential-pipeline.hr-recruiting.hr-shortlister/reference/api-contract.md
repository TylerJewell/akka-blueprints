# API contract — hr-shortlister

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/applications` | `SubmitApplicationRequest` | `201 { applicationId }` | `ShortlistEndpoint` → `ApplicationEntity` |
| `GET` | `/api/applications` | — | `200 [ ApplicationRecord... ]` (newest-first) | `ShortlistEndpoint` ← `ApplicationView` |
| `GET` | `/api/applications/{id}` | — | `200 ApplicationRecord` / `404` | `ShortlistEndpoint` ← `ApplicationView` |
| `POST` | `/api/applications/{id}/decision` | `RecruiterDecisionRequest` | `204` | `ShortlistEndpoint` → `ShortlistingWorkflow` |
| `GET` | `/api/applications/sse` | — | `text/event-stream` | `ShortlistEndpoint` ← `ApplicationView` |
| `GET` | `/api/drift-reports` | — | `200 [ DriftReport... ]` | `ShortlistEndpoint` ← `DriftReportEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitApplicationRequest (request body)

```json
{
  "jobId": "swe-l3-2026",
  "resumeText": "Alex Jordan\nalex.jordan@example.com\n\nExperience: 5 years...\nSkills: Java, Kubernetes, PostgreSQL..."
}
```

### RecruiterDecisionRequest (request body)

```json
{
  "recruiterId": "rec-007",
  "decision": "SHORTLIST",
  "overrideRationale": null
}
```

`overrideRationale` is `null` when the recruiter approves the agent's decision as-is. When the recruiter overrides, `decision` may differ from the agent's `ShortlistDecision.decision` and `overrideRationale` is a non-empty string.

### ApplicationRecord (response body)

```json
{
  "applicationId": "a-3fb2c1...",
  "jobId": "swe-l3-2026",
  "profile": {
    "applicantName": "Alex Jordan",
    "email": "alex.jordan@example.com",
    "resumeText": "...",
    "skills": ["Java", "Kubernetes", "PostgreSQL"],
    "experienceYears": 5,
    "highestEducation": "B.Sc. Computer Science",
    "currentTitle": "Software Engineer",
    "age": "[REDACTED]",
    "gender": null,
    "nationality": null,
    "disabilityIndicator": null
  },
  "score": {
    "criterionScores": [
      {
        "criterionId": "java-backend",
        "criterionName": "Java backend experience",
        "score": 8,
        "justification": "5 years stated experience; Java listed as primary skill."
      }
    ],
    "overallScore": 76,
    "scoredAt": "2026-06-28T10:05:00Z"
  },
  "agentDecision": {
    "decision": "SHORTLIST",
    "rationale": "Java backend score (8/10) exceeds the L3 threshold of 6.",
    "confidenceScore": 82,
    "decidedAt": "2026-06-28T10:05:05Z"
  },
  "recruiterDecision": {
    "recruiterId": "rec-007",
    "decision": "SHORTLIST",
    "overrideRationale": null,
    "decidedAt": "2026-06-28T10:10:00Z"
  },
  "status": "WRITTEN",
  "redactions": [
    {
      "fieldName": "age",
      "originalPresence": "present",
      "redactedAt": "2026-06-28T10:04:58Z"
    }
  ],
  "receivedAt": "2026-06-28T10:04:50Z",
  "finishedAt": "2026-06-28T10:10:05Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `redactions` is an array — empty if no protected-attribute fields were present in the resume.

### DriftReport (response body)

```json
{
  "cohortDimension": "gender",
  "cohortA": "M",
  "cohortB": "F",
  "cohortARate": 0.62,
  "cohortBRate": 0.49,
  "ratio": 1.27,
  "threshold": 1.20,
  "breached": true,
  "computedAt": "2026-06-28T06:00:00Z"
}
```

### SSE event format

```
event: application-update
data: { "applicationId": "a-3fb2c1...", "status": "SCORED", "score": { ... }, ... }
```

One event per state transition (`RECEIVED`, `PARSING`, `PARSED`, `SCORING`, `SCORED`, `DECIDING`, `DECIDED`, `AWAITING_APPROVAL`, `APPROVED`, `WRITTEN`, `FAILED`) and one per `SpecialCategoryRedacted` audit event:

```
event: application-redaction
data: { "applicationId": "a-3fb2c1...", "fieldName": "age", "redactedAt": "..." }
```

Clients reconcile by `applicationId`; an event always carries the full row at the moment of transition.

## Authorization

ACL: open to the internet (local-dev only). A production deployment must restrict `POST /api/applications/{id}/decision` to authenticated recruiters and stamp the `recruiterId` from the verified principal rather than accepting it from the request body.
