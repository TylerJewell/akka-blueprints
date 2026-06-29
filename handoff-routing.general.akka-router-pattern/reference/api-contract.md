# API contract — akka-router-pattern

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/requests` | — | `200 [ Request... ]` (newest-first); optional `?domain=CONTENT|CODE|DATA|UNKNOWN&status=…` filtered client-side | `RouterEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 Request` / `404` | `RouterEndpoint` ← `RequestView` |
| `POST` | `/api/requests` | `{ "requesterId": String, "channel": String, "title": String, "body": String }` | `201 { "requestId": String }` | `RouterEndpoint` → `TaskQueue` |
| `POST` | `/api/requests/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Request` (status now `COMPLETED`) / `409` if not `BLOCKED` | `RouterEndpoint` → `RequestEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `RouterEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### TaskRequest (request body for `POST /api/requests`, minus `requestId` and `receivedAt`)

```json
{
  "requesterId": "user-42",
  "channel": "web-form",
  "title": "Write a product announcement email",
  "body": "We are launching a new storage tier on Friday. Please draft a 3-paragraph announcement for our mailing list. Tone: professional but friendly."
}
```

### Request

```json
{
  "requestId": "req-7a3c21e0",
  "incoming": {
    "requestId": "req-7a3c21e0",
    "requesterId": "user-42",
    "channel": "web-form",
    "title": "Write a product announcement email",
    "body": "We are launching a new storage tier on Friday...",
    "receivedAt": "2026-06-28T09:10:00Z"
  },
  "classification": {
    "domain": "CONTENT",
    "confidence": "high",
    "reason": "Explicit email drafting task with no code or data component."
  },
  "guardrail": {
    "verdict": "ALLOWED",
    "violations": [],
    "rubricVersion": "v1"
  },
  "result": {
    "resultTitle": "Product announcement email: new storage tier",
    "resultBody": "Subject: Introducing our new storage tier...\n\n...",
    "status": "COMPLETED",
    "specialistTag": "content",
    "completedAt": "2026-06-28T09:10:14Z"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Domain right; reason identifies the email-drafting signal directly.",
    "scoredAt": "2026-06-28T09:10:08Z"
  },
  "unroutableReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:14Z"
}
```

A blocked request has `guardrail.verdict = "DENIED"`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An unroutable request has `classification.domain = "UNKNOWN"`, `unroutableReason` populated, `status = "UNROUTABLE"`.

### ClassificationDecision (returned by `ClassifierAgent`, embedded in `Request.classification`)

```json
{ "domain": "CONTENT" | "CODE" | "DATA" | "UNKNOWN",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### GuardrailVerdict (returned by `RoutingGuardrail`, embedded in `Request.guardrail`)

```json
{ "verdict": "ALLOWED" | "DENIED",
  "violations": ["prompt-injection-detected", "cross-domain-confusion", …],
  "rubricVersion": "v1" }
```

### TaskResult (returned by a specialist, embedded in `Request.result`)

```json
{ "resultTitle": "short descriptive label",
  "resultBody": "full output text",
  "status": "COMPLETED" | "ESCALATED",
  "specialistTag": "content" | "code" | "data",
  "completedAt": "ISO-8601" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Request.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: request-update
data: { "requestId": "req-…", "status": "CLASSIFIED", … full Request JSON … }
```

One event per state transition on the `RequestView`. Clients reconcile by `requestId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/requests`).
- `404` — unknown `requestId`.
- `409` — `unblock` requested on a request not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
