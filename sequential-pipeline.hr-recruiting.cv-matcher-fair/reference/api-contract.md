# API contract — Fair CV Matcher

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Lifecycle fields are nullable — `Optional` in Java, serialized as the raw value or `null`.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/candidates` | `{ cvText, postings, slice }` | `{ candidateId }` | MatchingEndpoint → CvIntakeQueue |
| POST | `/api/candidates/{id}/review` | `{ reviewedBy, note }` | `200` \| `404` | MatchingEndpoint → CandidateEntity |
| GET | `/api/candidates` | — (opt `?status=`) | `{ candidates: [Candidate, ...] }` | MatchingEndpoint → MatchesView |
| GET | `/api/candidates/{id}` | — | `Candidate` | MatchingEndpoint → CandidateEntity |
| GET | `/api/candidates/sse` | — | SSE of `Candidate` | MatchingEndpoint → MatchesView |
| GET | `/api/fairness` | — | `{ slices: [SliceStat, ...] }` | MatchingEndpoint → MatchesView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | MatchingEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | MatchingEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | MatchingEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

`GET /api/candidates?status=` filters client-side from `getAllCandidates` (Akka cannot auto-index the enum column — Lesson 2).

## Payload shapes

Submission request:

```json
{ "cvText": "string", "postings": [ { "jobId": "string", "jobTitle": "string", "requirements": "string" } ], "slice": "string" }
```

Candidate:

```json
{
  "id": "uuid",
  "status": "SUBMITTED | EXTRACTED | SANITIZED | MATCHED | REVIEWED",
  "submittedAt": "ISO-8601",
  "slice": "string or null",
  "profile": { "yearsExperience": 0, "skills": ["string"], "education": "string", "currentTitle": "string" },
  "redactions": [ { "field": "string", "category": "string" } ],
  "matches": [ { "jobId": "string", "jobTitle": "string", "score": 0, "rationale": "string" } ],
  "extractedAt": "ISO-8601 or null",
  "sanitizedAt": "ISO-8601 or null",
  "matchedAt": "ISO-8601 or null",
  "reviewedAt": "ISO-8601 or null",
  "reviewedBy": "string or null",
  "reviewNote": "string or null"
}
```

SliceStat:

```json
{ "slice": "string", "matchedCount": 0, "selectionRate": 0.0, "parityRatio": 0.0, "flagged": false }
```

## SSE event format

`GET /api/candidates/sse` emits `text/event-stream`. Each event `data:` line is one serialized `Candidate` JSON object, pushed on every entity change. No custom event names; the client parses each `data:` payload as a candidate and upserts by `id`.
