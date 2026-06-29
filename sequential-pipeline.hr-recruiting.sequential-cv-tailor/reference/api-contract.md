# API contract — sequential-cv-tailor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/cv-requests` | `SubmitCvRequest` | `201 { requestId }` | `CvEndpoint` → `CvRequestEntity` |
| `GET` | `/api/cv-requests` | — | `200 [ CvRequestRecord... ]` (newest-first) | `CvEndpoint` ← `CvRequestView` |
| `GET` | `/api/cv-requests/{id}` | — | `200 CvRequestRecord` / `404` | `CvEndpoint` ← `CvRequestView` |
| `GET` | `/api/cv-requests/sse` | — | `text/event-stream` | `CvEndpoint` ← `CvRequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `CvEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `CvEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `CvEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCvRequest (request body)

```json
{
  "candidateProfileId": "alice-chen",
  "jobPostingId": "senior-java-engineer"
}
```

### CvRequestRecord (response body)

```json
{
  "requestId": "r-3c7a91...",
  "candidateProfileId": "alice-chen",
  "jobPostingId": "senior-java-engineer",
  "baseCv": {
    "generatedSummary": "Experienced Java engineer with 8 years delivering distributed systems across fintech and e-commerce. Strong background in Spring Boot microservices and event-driven architecture. Proven track record of leading small cross-functional teams to production delivery.",
    "skills": [
      { "name": "Java", "level": "expert" },
      { "name": "Spring Boot", "level": "expert" },
      { "name": "Apache Kafka", "level": "intermediate" }
    ],
    "experience": [
      {
        "company": "Meridian Payments",
        "title": "Senior Software Engineer",
        "period": "2021–present",
        "bulletPoints": "Designed and deployed a Kafka-backed event bus handling 80k events/day.\nMigrated three legacy SOAP services to REST, reducing error rate by 40%.\nMentored two junior engineers through onboarding and first production deployments."
      }
    ],
    "generatedAt": "2026-06-28T10:00:10Z"
  },
  "tailoredCv": {
    "tailoredSummary": "Senior Java Engineer with 8 years of hands-on experience building distributed systems, specialising in Spring Boot and Apache Kafka. Delivered event-driven microservices at Meridian Payments and led cross-functional delivery teams.",
    "skills": [
      { "name": "Java", "level": "expert" },
      { "name": "Spring Boot", "level": "expert" },
      { "name": "Apache Kafka", "level": "intermediate" }
    ],
    "experience": [
      {
        "company": "Meridian Payments",
        "title": "Senior Software Engineer",
        "period": "2021–present",
        "bulletPoints": "Architected a Spring Boot microservice pipeline backed by Apache Kafka, processing 80k events/day.\nMigrated three SOAP services to REST APIs, reducing error rate by 40%.\nMentored junior engineers; led two production deployments end-to-end."
      }
    ],
    "keywordMatches": [
      { "keyword": "Java", "required": true, "sourceSection": "skills" },
      { "keyword": "Spring Boot", "required": true, "sourceSection": "experience" },
      { "keyword": "Apache Kafka", "required": false, "sourceSection": "experience" },
      { "keyword": "Kubernetes", "required": true, "sourceSection": null }
    ],
    "tailoredAt": "2026-06-28T10:00:25Z"
  },
  "alignmentResult": {
    "score": 3,
    "rationale": "Required keyword missing: Kubernetes not found in summary or experience.",
    "keywordsCovered": ["Java", "Spring Boot"],
    "keywordsMissed": ["Kubernetes"],
    "scoredAt": "2026-06-28T10:00:26Z"
  },
  "sanitizationCount": 1,
  "status": "SCORED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:26Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (`Optional.empty()` serialises as JSON `null`). `sanitizationCount` is 0 if no model calls have fired yet; it increments on every `SanitizationApplied` event.

### SSE event format

```
event: cv-request-update
data: { "requestId": "r-3c7a91...", "status": "TAILORED", "tailoredCv": { ... }, ... }
```

One event per state transition (`CREATED`, `GENERATING`, `GENERATED`, `TAILORING`, `TAILORED`, `SCORED`, `FAILED`) and one per `SanitizationApplied` audit event:

```
event: sanitization-applied
data: { "requestId": "r-3c7a91...", "agent": "CvGeneratorAgent", "removedCount": 3, "appliedAt": "2026-06-28T10:00:02Z" }
```

Clients reconcile by `requestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay earlier events.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitCvRequest` record and the `CvRequestCreated` event to carry it.
