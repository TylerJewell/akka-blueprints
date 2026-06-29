# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/requisitions` | `Requisition` | `{ candidateIds: [string] }` | `RecruitmentEndpoint` |
| GET | `/api/candidates` | — | `{ candidates: [Candidate] }` | `RecruitmentEndpoint` |
| GET | `/api/candidates/{id}` | — | `Candidate` \| 404 | `RecruitmentEndpoint` |
| GET | `/api/candidates/sse` | — | SSE of `Candidate` | `RecruitmentEndpoint` |
| POST | `/api/system/halt` | — | `{ halted: true }` | `RecruitmentEndpoint` |
| POST | `/api/system/resume` | — | `{ halted: false }` | `RecruitmentEndpoint` |
| GET | `/api/system/status` | — | `{ halted: boolean }` | `RecruitmentEndpoint` |
| GET | `/api/monitoring` | — | `MonitoringSnapshot` | `RecruitmentEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `RecruitmentEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `RecruitmentEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `RecruitmentEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

```json
// Requisition (request body of POST /api/requisitions)
{
  "roleTitle": "Senior Backend Engineer",
  "requirements": "5+ yrs JVM, distributed systems, event sourcing",
  "candidateCount": 3,
  "forceBlockedDomain": false
}
```

```json
// Candidate (lifecycle fields are null until their stage fires)
{
  "id": "uuid",
  "requisitionId": "uuid or null",
  "roleTitle": "string or null",
  "status": "SOURCED | SANITIZED | SCREENED | MATCHED | REJECTED | BLOCKED | HALTED",
  "sourcedAt": "ISO-8601 or null",
  "sourceHandle": "string or null",
  "sanitizedAt": "ISO-8601 or null",
  "redactedCategories": "string or null",
  "screenedAt": "ISO-8601 or null",
  "screenSummary": "string or null",
  "screenPass": "boolean or null",
  "matchedAt": "ISO-8601 or null",
  "matchScore": "number or null",
  "matchRationale": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectReason": "string or null",
  "blockedAt": "ISO-8601 or null",
  "blockedDomain": "string or null",
  "haltedAt": "ISO-8601 or null"
}
```

```json
// MonitoringSnapshot (GET /api/monitoring)
{
  "evaluated": 12,
  "matched": 7,
  "rejected": 3,
  "blocked": 2,
  "avgScore": 64.5
}
```

## SSE event format

`GET /api/candidates/sse` streams `text/event-stream`. Each event's `data:` line
is one `Candidate` JSON object (the form above), emitted on every
`CandidateEntity` event. The UI upserts by `id`.
