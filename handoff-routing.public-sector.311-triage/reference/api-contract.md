# API contract — 311-triage

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/requests` | — | `200 [ ServiceRequest... ]` (newest-first); optional `?category=…&status=…` filtered client-side | `TriageEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 ServiceRequest` / `404` | `TriageEndpoint` ← `RequestView` |
| `POST` | `/api/requests` | `{ "constituentId": String, "channel": String, "subject": String, "description": String, "locationHint": String }` | `201 { "requestId": String }` | `TriageEndpoint` → `RequestQueue` |
| `POST` | `/api/requests/{id}/review` | `{ "reviewedBy": String, "decision": "approve" \| "escalate", "note": String }` | `200 ServiceRequest` / `409` if not `FLAGGED_FOR_REVIEW` | `TriageEndpoint` → `ServiceRequestEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `TriageEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundServiceRequest (request body for `POST /api/requests`, minus `requestId` and `receivedAt`)

```json
{
  "constituentId": "cst-4421",
  "channel": "web-form",
  "subject": "Pothole on Oak Street near the library",
  "description": "There is a large pothole at the corner of Oak St and 5th Ave. Cars are swerving to avoid it. My name is Jane Smith and I can be reached at 555-123-4567.",
  "locationHint": "Oak Street at 5th Ave"
}
```

### ServiceRequest

```json
{
  "requestId": "req-7a3f19dc",
  "inbound": {
    "requestId": "req-7a3f19dc",
    "constituentId": "cst-4421",
    "channel": "web-form",
    "subject": "Pothole on Oak Street near the library",
    "description": "...raw with PII...",
    "locationHint": "Oak Street at 5th Ave",
    "receivedAt": "2026-06-28T09:00:00Z"
  },
  "sanitized": {
    "redactedSubject": "Pothole on Oak Street near the library",
    "redactedDescription": "There is a large pothole at the corner of Oak St and 5th Ave. Cars are swerving to avoid it. My name is [REDACTED-NAME] and I can be reached at [REDACTED-PHONE].",
    "redactedLocation": "Oak Street at 5th Ave",
    "piiCategoriesFound": ["name", "phone"]
  },
  "triage": {
    "category": "PUBLIC_WORKS",
    "confidence": "high",
    "reason": "Road surface defect reported on a public street."
  },
  "routeVerdict": {
    "approved": true,
    "flags": [],
    "rubricVersion": "v1"
  },
  "response": {
    "responseSubject": "Re: Pothole on Oak Street near the library",
    "responseBody": "Thank you for reporting the road surface issue at Oak Street and 5th Avenue. A work order has been opened...",
    "action": "WORK_ORDER_CREATED",
    "departmentTag": "public-works",
    "respondedAt": "2026-06-28T09:00:22Z"
  },
  "triageScore": {
    "score": 5,
    "rationale": "Category right; reason names the infrastructure signal directly.",
    "scoredAt": "2026-06-28T09:00:12Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:23Z"
}
```

A flagged request has `routeVerdict.approved = false`, `routeVerdict.flags` non-empty, `status = "FLAGGED_FOR_REVIEW"`, `finishedAt = null`. An escalated request has `triage.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### TriageDecision (returned by `TriageAgent`, embedded in `ServiceRequest.triage`)

```json
{ "category": "PUBLIC_WORKS" | "PERMITS_ZONING" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### RouteVerdict (returned by `RouteGuardrail`, embedded in `ServiceRequest.routeVerdict`)

```json
{ "approved": true | false,
  "flags": ["low-confidence-route", "content-category-mismatch", "out-of-jurisdiction"],
  "rubricVersion": "v1" }
```

### DepartmentResponse (returned by either specialist, embedded in `ServiceRequest.response`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "2–4 short paragraphs",
  "action": "WORK_ORDER_CREATED" | "PERMIT_INFO_PROVIDED" | "INSPECTION_SCHEDULED" | "REFERRAL_ISSUED" | "INFO_PROVIDED" | "ESCALATED",
  "departmentTag": "public-works" | "permits-zoning",
  "respondedAt": "ISO-8601" }
```

### TriageScore (returned by `TriageJudge`, embedded in `ServiceRequest.triageScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: request-update
data: { "requestId": "req-…", "status": "TRIAGED", … full ServiceRequest JSON … }
```

One event per state transition on the `RequestView`. Clients reconcile by `requestId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/requests`).
- `404` — unknown `requestId`.
- `409` — `review` requested on a request not currently in `FLAGGED_FOR_REVIEW`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
