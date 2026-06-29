# API contract — `recruitment-team`

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Served by `ScreeningEndpoint` (`/api`) and `AppEndpoint` (static).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/applications` | `{ "roleId": string, "resume": string }` | `{ "candidateId": string }` | `ScreeningEndpoint` → `InboundApplicationQueue` |
| POST | `/api/candidates/{id}/approve` | `{ "decidedBy": string }` | `200` \| `404` | `ScreeningEndpoint` → `CandidateEntity` |
| POST | `/api/candidates/{id}/reject` | `{ "decidedBy": string, "reason": string }` | `200` \| `404` | `ScreeningEndpoint` → `CandidateEntity` |
| GET | `/api/candidates` | — | `{ "candidates": [Candidate, ...] }` | `ScreeningEndpoint` → `CandidatesView` |
| GET | `/api/candidates/{id}` | — | `Candidate` | `ScreeningEndpoint` → `CandidatesView` |
| GET | `/api/candidates/sse` | — | SSE stream of `Candidate` | `ScreeningEndpoint` → `CandidatesView` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `ScreeningEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `ScreeningEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `ScreeningEndpoint` |
| GET | `/` | — | `302 /app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` |

The candidates list filters client-side in the endpoint from `getAllCandidates`; the view has no `WHERE status` query (Akka cannot auto-index enum columns, Lesson 2).

## Candidate JSON

Lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire (Lesson 6).

```json
{
  "id": "uuid",
  "roleId": "string",
  "sanitizedResume": "string or null",
  "status": "RECEIVED | SANITIZED | MATCHED | SCREENED | AWAITING_DECISION | APPROVED | REJECTED | ESCALATED",
  "matchScore": "int or null",
  "matchSummary": "string or null",
  "screenRecommendation": "ADVANCE | HOLD | REJECT or null",
  "screenReasons": "string or null",
  "evalScore": "int or null",
  "supervisorSummary": "string or null",
  "decidedBy": "string or null",
  "decisionReason": "string or null",
  "sanitizedAt": "ISO-8601 or null",
  "screenedAt": "ISO-8601 or null",
  "decidedAt": "ISO-8601 or null",
  "escalatedAt": "ISO-8601 or null",
  "driftFlagged": "bool or null"
}
```

## SSE event format

`GET /api/candidates/sse` emits one `data:` line per updated `Candidate`, JSON-encoded as above, terminated by a blank line. The UI subscribes with `EventSource` and upserts each candidate into the list by `id`.
