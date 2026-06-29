# API contract — basic-cv-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/cv-requests` | `SubmitCvRequest` | `201 { requestId }` | `CvEndpoint` → `CvEntity` |
| `GET` | `/api/cv-requests` | — | `200 [ CvRequestState... ]` (newest-first) | `CvEndpoint` ← `CvView` |
| `GET` | `/api/cv-requests/{id}` | — | `200 CvRequestState` / `404` | `CvEndpoint` ← `CvView` |
| `GET` | `/api/cv-requests/sse` | — | `text/event-stream` | `CvEndpoint` ← `CvView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCvRequest (request body)

```json
{
  "candidateFullName": "Alex Rivera",
  "contactEmail": "alex.rivera@personal.example",
  "targetRole": "Senior Software Engineer",
  "workHistory": [
    {
      "title": "Backend Engineer",
      "company": "Acme Retail",
      "startDate": "2021-03",
      "endDate": "2026-01",
      "bullets": [
        "Designed a Python ETL pipeline processing 2 M daily events",
        "Reduced API p99 latency from 480 ms to 120 ms"
      ]
    }
  ],
  "education": [
    {
      "degree": "BSc Computer Science",
      "institution": "State University",
      "graduationYear": "2020"
    }
  ],
  "skills": ["Python", "Java", "PostgreSQL", "Docker"],
  "additionalNotes": "National Insurance: AB123456C. Available from July 2026.",
  "outputMode": "PROSE",
  "submittedBy": "recruiter-007"
}
```

### CvRequestState (response body)

```json
{
  "requestId": "cv-5f3a...",
  "request": {
    "requestId": "cv-5f3a...",
    "candidateFullName": "Alex Rivera",
    "contactEmail": "alex.rivera@personal.example",
    "targetRole": "Senior Software Engineer",
    "workHistory": [ { "title": "Backend Engineer", "company": "Acme Retail", "startDate": "2021-03", "endDate": "2026-01", "bullets": ["..."] } ],
    "education": [ { "degree": "BSc Computer Science", "institution": "State University", "graduationYear": "2020" } ],
    "skills": ["Python", "Java", "PostgreSQL", "Docker"],
    "additionalNotes": "National Insurance: AB123456C. Available from July 2026.",
    "outputMode": "PROSE",
    "submittedBy": "recruiter-007",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "sanitized": {
    "redactedContactEmail": "[REDACTED-EMAIL]",
    "redactedAdditionalNotes": "National Insurance: [REDACTED-NINO]. Available from July 2026.",
    "piiCategoriesFound": ["email", "nino"]
  },
  "generatedCv": {
    "headline": "Software Engineer — Python and Java backend systems",
    "sections": [
      {
        "sectionTitle": "Professional Summary",
        "content": "Software engineer with five years of experience building data pipelines and REST APIs..."
      },
      {
        "sectionTitle": "Work History",
        "content": "### Backend Engineer — Acme Retail (2021–2026)\n- Designed and shipped a Python ETL pipeline..."
      },
      {
        "sectionTitle": "Education",
        "content": "### BSc Computer Science — State University (2020)"
      },
      {
        "sectionTitle": "Skills",
        "content": "Python, Java, PostgreSQL, Docker, REST API design, data pipeline development"
      }
    ],
    "keywords": ["Python", "Java", "ETL", "REST API", "PostgreSQL", "Docker", "microservices", "backend"],
    "outputMode": "PROSE",
    "generatedAt": "2026-06-28T14:00:18Z"
  },
  "status": "CV_GENERATED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:18Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: cv-update
data: { "requestId": "cv-5f3a...", "status": "CV_GENERATED", "generatedCv": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `GENERATING`, `CV_GENERATED`, `FAILED`). Clients reconcile by `requestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
