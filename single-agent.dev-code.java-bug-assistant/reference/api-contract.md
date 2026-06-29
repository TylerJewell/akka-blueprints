# API contract — java-bug-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/bugs` | `SubmitBugRequest` | `201 { bugId }` | `BugEndpoint` → `BugEntity` |
| `GET` | `/api/bugs` | — | `200 [ Bug... ]` (newest-first) | `BugEndpoint` ← `BugView` |
| `GET` | `/api/bugs/{id}` | — | `200 Bug` / `404` | `BugEndpoint` ← `BugView` |
| `GET` | `/api/bugs/sse` | — | `text/event-stream` | `BugEndpoint` ← `BugView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitBugRequest (request body)

```json
{
  "title": "NPE in payment service order processor",
  "rawReport": "java.lang.NullPointerException at com.example.payment.OrderProcessor.process(OrderProcessor.java:47)\n  DB_PASSWORD=s3cr3t\n  host: db-prod-01.internal\n  ...",
  "category": "RUNTIME_ERROR",
  "submittedBy": "engineer-42"
}
```

`category` is optional; omit or set to `"UNKNOWN"` to trigger auto-detect in `BugNormalizer`.

### Bug (response body)

```json
{
  "bugId": "b-9f3...",
  "report": {
    "bugId": "b-9f3...",
    "title": "NPE in payment service order processor",
    "rawReport": "(raw report preserved for audit)",
    "category": "RUNTIME_ERROR",
    "submittedBy": "engineer-42",
    "submittedAt": "2026-06-28T14:22:00Z"
  },
  "normalized": {
    "cleanedReport": "java.lang.NullPointerException at com.example.payment.OrderProcessor.process(OrderProcessor.java:47)\n  DB_PASSWORD=[REDACTED-CREDENTIAL]\n  host: [REDACTED-HOST]\n  ...",
    "redactionCategories": ["credential", "internal-hostname"]
  },
  "searchResults": [
    {
      "ticketId": "TICKET-042",
      "summary": "NPE in OrderProcessor when customer session expires",
      "ticketStatus": "RESOLVED",
      "resolution": "Added null-check guard before getCustomerId() call in OrderProcessor.process()"
    },
    {
      "ticketId": "TICKET-107",
      "summary": "Intermittent NPE in payment flow under high load",
      "ticketStatus": "OPEN",
      "resolution": ""
    }
  ],
  "recommendation": {
    "action": "RESOLVED",
    "rationale": "The stack trace matches the pattern in TICKET-042, resolved by a null-check guard in the same method and line. TICKET-107 covers a related path but remains open. The high-confidence fix from TICKET-042 applies directly.",
    "candidateFixes": [
      {
        "fixId": "fix-1",
        "confidenceScore": 85,
        "description": "Apply null-check guard to order processor before calling getCustomerId(), as resolved in TICKET-042.",
        "sourceRef": "TICKET-042"
      },
      {
        "fixId": "fix-2",
        "confidenceScore": 40,
        "description": "Investigate whether the null input originates in the upstream session handler referenced in TICKET-107.",
        "sourceRef": "TICKET-107"
      }
    ],
    "decidedAt": "2026-06-28T14:22:18Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Every fix cites a source ticket; rationale is actionable; one fix has low confidence which is appropriate given the open ticket.",
    "evaluatedAt": "2026-06-28T14:22:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T14:22:00Z",
  "finishedAt": "2026-06-28T14:22:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: bug-update
data: { "bugId": "b-9f3...", "status": "RECOMMENDATION_RECORDED", "recommendation": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `NORMALIZED`, `SEARCHING`, `SEARCH_COMPLETE`, `RESOLVING`, `RECOMMENDATION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `bugId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
