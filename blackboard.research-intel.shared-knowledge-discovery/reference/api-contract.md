# API contract

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | /api/inquiries | `{ "question": String, "submittedBy"?: String }` | 202 `{ "inquiryId": String }` | KnowledgeEndpoint → IntakeQueue |
| GET | /api/inquiries/{id} | — | 200 Inquiry / 404 | KnowledgeEndpoint ← InquiryEntity |
| GET | /api/blackboards/{id} | — | 200 Blackboard / 404 | KnowledgeEndpoint ← BlackboardEntity |
| GET | /api/blackboards | — | 200 [ BlackboardRow... ] | KnowledgeEndpoint ← BlackboardView |
| GET | /api/blackboards?status=… | — | 200 [ BlackboardRow... ] (filtered client-side) | KnowledgeEndpoint ← BlackboardView |
| GET | /api/blackboards/sse | — | text/event-stream (one event per blackboard change) | KnowledgeEndpoint ← BlackboardView |
| POST | /api/inquiries/{id}/close | `{ "reason": String }` | 200 Inquiry | KnowledgeEndpoint → InquiryEntity |
| POST | /api/control/halt | `{ "reason": String, "by": String }` | 200 SystemControl | KnowledgeEndpoint → SystemControl |
| POST | /api/control/resume | — | 200 SystemControl | KnowledgeEndpoint → SystemControl |
| GET | /api/control | — | 200 SystemControl | KnowledgeEndpoint ← SystemControl |
| GET | /api/metadata/readme | — | text/markdown | AppEndpoint |
| GET | /api/metadata/risk-survey | — | text/yaml | AppEndpoint |
| GET | /api/metadata/eval-matrix | — | text/yaml | AppEndpoint |
| GET | / | — | 302 → /app/index.html | AppEndpoint |
| GET | /app/* | — | static UI (single self-contained index.html) | AppEndpoint |

## JSON payloads

**Inquiry:**
```json
{
  "inquiryId": "i-7c1",
  "question": "What is the current consensus on room-temperature superconductors?",
  "submittedBy": "ui",
  "status": "ACTIVE",
  "reportSummary": null,
  "createdAt": "2026-06-29T09:14:02Z",
  "completedAt": null
}
```

**Blackboard:**
```json
{
  "inquiryId": "i-7c1",
  "question": "What is the current consensus on room-temperature superconductors?",
  "status": "OPEN",
  "sources": [ { "sourceId": "s-1", "citation": "...", "url": "...", "addedAt": "..." } ],
  "claims": [ { "claimId": "i-7c1:c:0", "text": "...", "supports": ["..."], "derivedFrom": "s-1", "confidence": 0.72 } ],
  "hypotheses": [ { "hypothesisId": "i-7c1:h:0", "statement": "...", "backedByClaims": ["i-7c1:c:0"], "proposedAt": "..." } ],
  "disputed": [],
  "openSubQuestions": ["What ambient pressure has been demonstrated?"],
  "iterationCount": 4,
  "report": null,
  "lastWriteAt": "2026-06-29T09:14:55Z",
  "createdAt": "2026-06-29T09:14:02Z"
}
```

Lifecycle fields on `Blackboard` and `Inquiry` (`report`, `lastWriteAt`, `reportSummary`, `completedAt`) are `Optional<T>` in Java and serialise as raw value or null (Lesson 6).

**SystemControl:**
```json
{ "halted": false, "haltedReason": null, "haltedBy": null, "haltedAt": null }
```

**SSE event format:**
```
event: blackboard-update
data: { "inquiryId": "i-7c1", "status": "OPEN", "iterationCount": 4, ... }
```

One event per blackboard state transition. Clients reconcile by `inquiryId` and re-render the per-inquiry panel.
