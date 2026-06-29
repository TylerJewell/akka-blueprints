# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/hiring` | `{ "candidateId": "string", "roleId": "string", "resumeText": "string", "availabilityWindow": "string" }` | `{ "applicationId": "uuid" }` | `HiringEndpoint` → `ApplicationQueue` |
| GET | `/api/hiring` | — | `{ "applications": [ApplicationRow, ...] }` | `HiringEndpoint` → `ApplicationView` |
| GET | `/api/hiring?status=RECOMMENDED` | — | filtered list (client-side filter) | `HiringEndpoint` |
| GET | `/api/hiring/{id}` | — | `ApplicationRow` or 404 | `HiringEndpoint` |
| GET | `/api/hiring/sse` | — | `text/event-stream` of `ApplicationRow` | `HiringEndpoint` → `ApplicationView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `HiringEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `HiringEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `HiringEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/hiring` request:

```json
{
  "candidateId": "cand-001",
  "roleId": "eng-senior-backend",
  "resumeText": "10 years Java, led distributed systems team at ...",
  "availabilityWindow": "weekdays after 14:00 UTC, next 10 business days"
}
```

`ApplicationRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "applicationId": "uuid",
  "candidateId": "string",
  "roleId": "string",
  "status": "RECEIVED | UNDER_REVIEW | RECOMMENDED | REJECTED | DEGRADED | POLICY_HOLD",
  "screeningReport": {
    "candidateId": "string",
    "qualificationScore": 82,
    "strengthHighlights": ["..."],
    "gapNotes": ["..."],
    "screenedAt": "ISO-8601"
  },
  "schedulingProposal": {
    "candidateId": "string",
    "slots": [
      { "slotId": "slot-1", "startTime": "ISO-8601", "endTime": "ISO-8601", "interviewFormat": "video" }
    ],
    "proposedAt": "ISO-8601"
  },
  "recommendation": {
    "decision": "RECOMMENDED",
    "screeningScore": 82,
    "proposedSlots": [{ "slotId": "slot-1", "startTime": "ISO-8601", "endTime": "ISO-8601", "interviewFormat": "video" }],
    "rationale": "string",
    "guardrailVerdict": "ok",
    "consolidatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/hiring/sse` emits one event per application change:

```
event: application
data: { ...ApplicationRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `applicationId`. No polling.
