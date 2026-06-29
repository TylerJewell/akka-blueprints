# API contract — auto-insurance-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/requests` | — | `200 [ MemberRequest... ]` (newest-first); supports optional `?category=…&status=…` filtered client-side | `InsuranceEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 MemberRequest` / `404` | `InsuranceEndpoint` ← `RequestView` |
| `POST` | `/api/requests` | `{ "memberId": String, "channel": String, "subject": String, "body": String }` | `201 { "requestId": String }` | `InsuranceEndpoint` → `RequestQueue` |
| `POST` | `/api/requests/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 MemberRequest` (status now `RESOLVED`) / `409` if not `BLOCKED` | `InsuranceEndpoint` → `RequestEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `InsuranceEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingRequest (request body for `POST /api/requests`, minus `requestId` and `receivedAt`)

```json
{
  "memberId": "MBR-[REDACTED]",
  "channel": "mobile-app",
  "subject": "Filing a claim for this morning's accident",
  "body": "I was in a collision on Route 9 this morning and need to start the claims process. My policy number is POL-[REDACTED]."
}
```

### MemberRequest

```json
{
  "requestId": "ins-4a71c8d2",
  "incoming": {
    "requestId": "ins-4a71c8d2",
    "memberId": "MBR-00441",
    "channel": "mobile-app",
    "subject": "Filing a claim for this morning's accident",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T09:10:00Z"
  },
  "sanitized": {
    "redactedSubject": "Filing a claim for this morning's accident",
    "redactedBody": "I was in a collision on Route 9 this morning and need to start the claims process. My policy number is [REDACTED-POLICY-NUMBER].",
    "piiCategoriesFound": ["member-id", "policy-number"]
  },
  "triage": {
    "category": "CLAIM",
    "confidence": "high",
    "reason": "Explicit first-notice-of-loss for a collision."
  },
  "response": {
    "responseSubject": "Re: Filing a claim for this morning's accident",
    "responseBody": "Your accident report has been received and a claim reference has been created...",
    "action": "CLAIM_ACKNOWLEDGED",
    "specialistTag": "claims",
    "confirmationRef": null,
    "resolvedAt": "2026-06-28T09:10:22Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "triageScore": {
    "score": 5,
    "rationale": "Category right; reason identifies the FNOL collision signal directly.",
    "scoredAt": "2026-06-28T09:10:18Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:23Z"
}
```

A blocked request has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated request has `triage.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### TriageDecision (returned by `ClaimTriageAgent`, embedded in `MemberRequest.triage`)

```json
{ "category": "CLAIM" | "POLICY" | "REWARDS" | "ROADSIDE" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### MemberResponse (returned by a specialist, embedded in `MemberRequest.response`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "2–5 short paragraphs",
  "action": "CLAIM_ACKNOWLEDGED" | "CLAIM_STATUS_PROVIDED" | "POLICY_INFO_PROVIDED" |
             "COVERAGE_CONFIRMED" | "REWARDS_REDEEMED" | "REWARDS_INFO_PROVIDED" |
             "ROADSIDE_DISPATCHED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "claims" | "policy" | "rewards" | "roadside",
  "confirmationRef": "RSA-K3F7M2QA" | null,
  "resolvedAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `ResponseGuardrail`, embedded in `MemberRequest.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-settlement-amount", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

### TriageScore (returned by `TriageJudge`, embedded in `MemberRequest.triageScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: request-update
data: { "requestId": "ins-…", "status": "TRIAGED", … full MemberRequest JSON … }
```

One event per state transition on the `RequestView`. Clients reconcile by `requestId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/requests`).
- `404` — unknown `requestId`.
- `409` — `unblock` requested on a request not currently in `BLOCKED`.
- `503` — backend unavailable (e.g. workflow timed out, recovery in progress).
