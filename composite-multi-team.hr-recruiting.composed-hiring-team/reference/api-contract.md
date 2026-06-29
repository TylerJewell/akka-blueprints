# API contract — composed-hiring-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/applications` | `{ "candidateName": String, "jobRole": String, "cvText": String, "requestedBy"?: String }` | `202 { "applicationId": String }` | `HiringEndpoint` → `ApplicationQueue` |
| `GET` | `/api/applications` | — | `200 [ Application... ]` | `HiringEndpoint` ← `ApplicationBoardView` |
| `GET` | `/api/applications?status=…` | — | `200 [ Application... ]` (filtered client-side) | `HiringEndpoint` ← `ApplicationBoardView` |
| `GET` | `/api/applications/{id}` | — | `200 Application` / `404` | `HiringEndpoint` ← `ApplicationEntity` |
| `GET` | `/api/applications/sse` | — | `text/event-stream` (one event per application change) | `HiringEndpoint` ← `ApplicationBoardView` |
| `GET` | `/api/applications/{id}/cv-iterations` | — | `200 CvIteration` | `HiringEndpoint` ← `CvBoardView` |
| `GET` | `/api/cv/sse` | — | `text/event-stream` (one event per CV draft change) | `HiringEndpoint` ← `CvBoardView` |
| `POST` | `/api/applications/{id}/post-hire-review` | `{ "reviewedBy": String, "outcome": "CLEARED"\|"FLAGGED", "comments": String }` | `200 Application` / `409` (application not `OFFER_EXTENDED`) / `404` | `HiringEndpoint` → `ApplicationEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON payloads

### Application

```json
{
  "applicationId": "app-9f2…",
  "candidateName": "Alice Chen",
  "jobRole": "Senior Software Engineer",
  "cvText": "5 years Java, distributed systems, Kubernetes CI/CD…",
  "status": "OFFER_EXTENDED",
  "hiringBrief": {
    "roleSummary": "A senior individual contributor building and operating distributed backend services.",
    "requiredDimensions": ["relevant engineering experience", "technical depth", "communication"],
    "targetCompetencies": ["problem-solving", "system design", "ownership", "collaboration"]
  },
  "screeningReport": {
    "summary": "The candidate demonstrates strong alignment across all three dimensions.",
    "overallOutcome": "PASS"
  },
  "acceptedCvDraft": {
    "iteration": 2,
    "changesApplied": ["added quantified metrics to role descriptions", "restructured skills section"]
  },
  "dimensionIds": ["app-9f2-d0", "app-9f2-d1", "app-9f2-d2"],
  "panelVerdict": { "outcome": "PROCEED", "reassessCompetencies": [] },
  "offerLetter": {
    "candidateName": "Alice Chen",
    "jobRole": "Senior Software Engineer",
    "offerReference": "OFFER-app-9f2"
  },
  "offerReference": "OFFER-app-9f2",
  "offerExtendedAt": "2026-06-28T10:30:15Z",
  "stageEvals": [
    { "stage": "screening", "score": 87, "flags": [], "evaluatedAt": "2026-06-28T10:22:10Z" },
    { "stage": "cv_improvement", "score": 82, "flags": [], "evaluatedAt": "2026-06-28T10:26:44Z" },
    { "stage": "interview", "score": 90, "flags": [], "evaluatedAt": "2026-06-28T10:29:55Z" }
  ],
  "postHireReview": null,
  "reassessCount": 0,
  "createdAt": "2026-06-28T10:18:00Z"
}
```

The `Application` returned by `GET /api/applications/{id}` carries the full `offerLetter.offerText` and `acceptedCvDraft.revisedCvText`; the board rows streamed over SSE drop those heavy fields (see `data-model.md`). Lifecycle fields (`hiringBrief`, `screeningReport`, `acceptedCvDraft`, `panelVerdict`, `offerLetter`, `offerReference`, `offerExtendedAt`, `postHireReview`) are `Optional<T>` in Java and serialise as the raw value or `null` (Lesson 6).

### CvIteration

```json
{
  "applicationId": "app-9f2…",
  "originalCvText": "5 years Java…",
  "drafts": [
    { "iteration": 1, "changesApplied": ["added project scope"], "revisedCvText": "…" },
    { "iteration": 2, "changesApplied": ["added quantified metrics", "restructured skills"], "revisedCvText": "…" }
  ],
  "acceptedDraft": { "iteration": 2, "changesApplied": ["added quantified metrics", "restructured skills"] },
  "latestCritique": { "iteration": 2, "outcome": "PASS", "fitScore": 85, "clarityScore": 82, "completenessScore": 88 },
  "iterationCount": 2,
  "createdAt": "2026-06-28T10:23:00Z"
}
```

The CV board row carries `iterationCount` and a `draftPresent` boolean rather than the full draft texts.

### PostHireReview (request and recorded form)

```json
{ "reviewedBy": "compliance-officer-1", "outcome": "FLAGGED", "comments": "Role description in offer letter does not match job posting; recommend correction before candidate start date." }
```

Recorded on the application as `postHireReview` with an added `reviewedAt`. Accepted only when the application status is `OFFER_EXTENDED`; otherwise the endpoint returns `409`.

### SSE event format

```
event: application-update
data: { "applicationId": "app-9f2…", "status": "INTERVIEWING", ... }

event: cv-update
data: { "applicationId": "app-9f2…", "iterationCount": 2, "draftPresent": true }
```

`GET /api/applications/sse` emits one `application-update` per application state transition; `GET /api/cv/sse` emits one `cv-update` per CV draft addition or acceptance. Clients reconcile applications by `applicationId` and group the CV board by iteration.
