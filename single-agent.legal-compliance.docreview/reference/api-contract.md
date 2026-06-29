# API contract — docreview

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/reviews` | `SubmitReviewRequest` | `201 { reviewId }` | `ReviewEndpoint` → `ReviewEntity` |
| `GET` | `/api/reviews` | — | `200 [ Review... ]` (newest-first) | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/reviews/{id}` | — | `200 Review` / `404` | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/reviews/sse` | — | `text/event-stream` | `ReviewEndpoint` ← `ReviewView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitReviewRequest (request body)

```json
{
  "documentTitle": "Vendor Y data-processing agreement (draft v3)",
  "rawDocument": "This Data Processing Addendum ('DPA') ...",
  "instructions": [
    {
      "instructionId": "dpa-sub-processor-disclosure",
      "text": "Document must list current sub-processors or reference a maintained list URL.",
      "severityFloor": "LOW"
    },
    {
      "instructionId": "dpa-breach-notification-sla",
      "text": "Document must specify a numeric breach-notification SLA, not 'without undue delay'.",
      "severityFloor": "MEDIUM"
    }
  ],
  "submittedBy": "compliance-reviewer-12"
}
```

### Review (response body)

```json
{
  "reviewId": "r-2bf...",
  "request": {
    "reviewId": "r-2bf...",
    "documentTitle": "Vendor Y data-processing agreement (draft v3)",
    "rawDocument": "(raw text preserved for audit)",
    "instructions": [ { "instructionId": "dpa-sub-processor-disclosure", "text": "...", "severityFloor": "LOW" } ],
    "submittedBy": "compliance-reviewer-12",
    "submittedAt": "2026-06-28T12:34:00Z"
  },
  "sanitized": {
    "redactedDocument": "This Data Processing Addendum ('DPA') between [REDACTED-COMPANY] and [REDACTED-COMPANY] ...",
    "piiCategoriesFound": ["company-name", "email", "address"]
  },
  "verdict": {
    "decision": "NEEDS_REVISION",
    "summary": "Sub-processor disclosure is satisfied; breach notification SLA is unspecified.",
    "findings": [
      {
        "instructionId": "dpa-sub-processor-disclosure",
        "severity": "LOW",
        "documentSection": "§3.2",
        "quote": "Processor maintains a current list of sub-processors at vendor.example/sub-processors.",
        "recommendation": "Reference the list URL in the recitals so a future revision cannot drop it without notice."
      }
    ],
    "decidedAt": "2026-06-28T12:34:18Z"
  },
  "eval": {
    "score": 4,
    "rationale": "Every finding cites a section and quote; one recommendation could be more action-oriented.",
    "evaluatedAt": "2026-06-28T12:34:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T12:34:00Z",
  "finishedAt": "2026-06-28T12:34:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: review-update
data: { "reviewId": "r-2bf...", "status": "VERDICT_RECORDED", "verdict": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `REVIEWING`, `VERDICT_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `reviewId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
