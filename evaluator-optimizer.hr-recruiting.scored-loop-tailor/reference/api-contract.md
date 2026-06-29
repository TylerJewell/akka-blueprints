# API contract — scored-loop-tailor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/applications` | `{ "jobTitle": String, "description"?: String, "candidateProfile"?: String }` | `202 { "applicationId": String }` | `RecruitingEndpoint` → `SubmissionQueue` |
| `GET` | `/api/applications` | — | `200 [ Application... ]` (optional `?status=TAILORING\|REVIEWING\|APPROVED\|REJECTED_FINAL` — filtered client-side from `getAllApplications`) | `RecruitingEndpoint` ← `ApplicationsView` |
| `GET` | `/api/applications/{id}` | — | `200 Application` / `404` | `RecruitingEndpoint` ← `ApplicationsView` |
| `GET` | `/api/applications/sse` | — | `text/event-stream` (one event per application change) | `RecruitingEndpoint` ← `ApplicationsView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/applications`

- Missing `description` → `""`.
- Missing `candidateProfile` → `""`.
- `jobTitle` is required; empty string returns `400`.
- Duplicate-detection window: 10 s on `(jobTitle, candidateProfile)`; the second submission returns the first `applicationId` (200) instead of starting a new workflow.

## JSON shapes

### Application

```json
{
  "applicationId": "app-4c1b9…",
  "jobTitle": "Senior Java Engineer",
  "description": "We are looking for a senior Java engineer…",
  "maxAttempts": 4,
  "status": "APPROVED",
  "attempts": [
    {
      "attemptNumber": 1,
      "draft": {
        "text": "Summary\nResults-driven Java engineer with eight years...\n\nExperience\n...\n\nSkills\n...",
        "sectionsPresent": ["Summary", "Experience", "Skills"],
        "draftedAt": "2026-06-28T10:01:02Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "REVISE",
        "notes": {
          "bullets": [
            "Experience section lacks quantified outcomes.",
            "Skills list does not include 'event-driven architecture'.",
            "Summary does not reference the target role title."
          ],
          "overallRationale": "Keyword alignment and clarity fall below threshold."
        },
        "score": 2,
        "reviewedAt": "2026-06-28T10:01:14Z"
      }
    },
    {
      "attemptNumber": 2,
      "draft": {
        "text": "Summary\nSenior Java engineer specialising in event-driven architecture...\n\nExperience\n- Reduced P99 latency by 40% on a 50k RPS pipeline...\n\nSkills\nJava, event-driven architecture, Akka, Kafka...",
        "sectionsPresent": ["Summary", "Experience", "Skills"],
        "draftedAt": "2026-06-28T10:01:21Z"
      },
      "guardrail": { "passed": true, "reasonCode": "OK", "detail": "" },
      "review": {
        "verdict": "APPROVE",
        "notes": { "bullets": [], "overallRationale": "All four dimensions score 4 or above." },
        "score": 5,
        "reviewedAt": "2026-06-28T10:01:33Z"
      }
    }
  ],
  "approvedAttemptNumber": 2,
  "approvedText": "Summary\nSenior Java engineer specialising in event-driven architecture…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T10:00:59Z",
  "finishedAt": "2026-06-28T10:01:34Z"
}
```

### Guardrail verdict forms

```json
{ "passed": true,  "reasonCode": "OK",               "detail": "" }
{ "passed": false, "reasonCode": "MISSING_SECTIONS",  "detail": "CV is missing required sections: Skills." }
```

`reasonCode` is one of `OK`, `MISSING_SECTIONS`. Additional codes can be added without breaking the contract; the UI renders any unknown code verbatim.

### SSE event format

```
event: application-update
data: { "applicationId": "app-4c1b9…", "status": "REVIEWING", "attempts": [...], ... }
```

One event per state transition. Clients reconcile by `applicationId`. The full `Application` JSON is included so a fresh client can render the row without a separate fetch.
