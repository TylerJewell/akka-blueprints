# API contract — parallel-hiring-reviewers

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/evaluations` | `{ "candidateName": String, "roleApplied": String, "resumeText": String, "submittedBy"?: String }` | `202 { "evaluationId": String }` | `ReviewEndpoint` → `CandidateQueue` |
| `GET` | `/api/evaluations` | — | `200 [ CandidateEvaluation... ]` | `ReviewEndpoint` ← `EvaluationView` |
| `GET` | `/api/evaluations?status=DECIDED` | — | `200 [ CandidateEvaluation... ]` (filtered client-side) | `ReviewEndpoint` ← `EvaluationView` |
| `GET` | `/api/evaluations/{id}` | — | `200 CandidateEvaluation` / `404` | `ReviewEndpoint` ← `EvaluationView` |
| `GET` | `/api/evaluations/sse` | — | `text/event-stream` (one event per evaluation change) | `ReviewEndpoint` ← `EvaluationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

The submitted `resumeText` is consumed by the workflow and sanitized at intake; it is never returned by any `GET` endpoint. Only the redacted résumé text and the reviewer findings are ever read back.

## JSON shapes

### CandidateEvaluation

```json
{
  "evaluationId": "ev-4a9…",
  "candidateName": "Jordan Smith",
  "roleApplied": "Senior Software Engineer",
  "status": "DECIDED",
  "redactedResumeText": "Experienced engineer with 8 years in distributed systems. Compensation expectation: $[REDACTED]. Previous role at [ORIGIN] office …",
  "redactionCount": 2,
  "hrReview": {
    "perspective": "HR",
    "decision": "ADVANCE",
    "score": 4,
    "findings": [ { "severity": "MINOR", "comment": "Compensation expectation slightly above posted band ceiling; clarification recommended." } ],
    "reviewedAt": "2026-06-28T09:14:01Z"
  },
  "managerReview": {
    "perspective": "MANAGER",
    "decision": "ADVANCE",
    "score": 5,
    "findings": [ { "severity": "INFO", "comment": "Distributed systems experience aligns well with all must-have requirements." } ],
    "reviewedAt": "2026-06-28T09:14:03Z"
  },
  "teamReview": {
    "perspective": "TEAM",
    "decision": "HOLD",
    "score": 3,
    "findings": [ { "severity": "CONCERN", "comment": "Profile emphasises independent contributions; team relies heavily on pair work." } ],
    "reviewedAt": "2026-06-28T09:14:04Z"
  },
  "recommendation": {
    "decision": "FURTHER_REVIEW",
    "rationale": "Two perspectives recommend advancing; the team perspective raised a CONCERN about working style that warrants a follow-up conversation before a final decision. HR flagged a minor compensation variance that is easily resolved in the offer stage …",
    "perspectiveReviews": [ { "perspective": "HR" }, { "perspective": "MANAGER" }, { "perspective": "TEAM" } ],
    "guardrailVerdict": "ok",
    "decidedAt": "2026-06-28T09:14:09Z"
  },
  "failureReason": null,
  "fairnessScore": 4,
  "fairnessRationale": "Recommendation correctly weights the CONCERN from the Team perspective; rationale references all three reviewers.",
  "createdAt": "2026-06-28T09:13:55Z",
  "finishedAt": "2026-06-28T09:14:09Z"
}
```

Lifecycle fields (`redactedResumeText`, `redactionCount`, `hrReview`, `managerReview`, `teamReview`, `recommendation`, `failureReason`, `fairnessScore`, `fairnessRationale`, `finishedAt`) are `Optional<T>` in Java and serialize as the raw value or `null` — never an `{ "value": … }` wrapper (Lesson 6).

### SSE event format

```
event: evaluation-update
data: { "evaluationId": "ev-4a9…", "status": "EVALUATING", ... }
```

One event per state transition. Clients reconcile by `evaluationId`. The `FairnessScored` transition emits an `evaluation-update` event whose status is unchanged but whose `fairnessScore` is now populated.
