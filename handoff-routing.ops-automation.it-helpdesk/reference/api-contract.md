# API contract — it-helpdesk

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/requests` | — | `200 [ Request... ]` (newest-first); optional `?category=…&status=…` filtered client-side | `HelpdeskEndpoint` ← `RequestView` |
| `GET` | `/api/requests/{id}` | — | `200 Request` / `404` | `HelpdeskEndpoint` ← `RequestView` |
| `POST` | `/api/requests` | `{ "submitterId": String, "channel": String, "subject": String, "body": String }` | `201 { "requestId": String }` | `HelpdeskEndpoint` → `RequestQueue` |
| `POST` | `/api/requests/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Request` (status now `RESOLVED`) / `409` if not `TICKET_BLOCKED` | `HelpdeskEndpoint` → `RequestEntity` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `HelpdeskEndpoint` ← `RequestView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingRequest (request body for `POST /api/requests`, minus `requestId` and `receivedAt`)

```json
{
  "submitterId": "emp-4472",
  "channel": "portal",
  "subject": "Locked out of Okta after password expired",
  "body": "Tried resetting via self-service but token already expired. Password: [REDACTED-SECRET]. Need urgent unlock."
}
```

### Request

```json
{
  "requestId": "req-9a1c44f7",
  "incoming": {
    "requestId": "req-9a1c44f7",
    "submitterId": "emp-4472",
    "channel": "portal",
    "subject": "Locked out of Okta after password expired",
    "body": "...raw with credentials...",
    "receivedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedSubject": "Locked out of Okta after password expired",
    "redactedBody": "Tried resetting via self-service but token already expired. Password: [REDACTED-SECRET]. Need urgent unlock.",
    "secretCategoriesFound": ["password"]
  },
  "classification": {
    "category": "ACCESS",
    "confidence": "high",
    "reason": "Account lockout on identity provider requires access-team action."
  },
  "resolution": {
    "responseSubject": "Re: Locked out of Okta after password expired",
    "responseBody": "Your Okta account has been unlocked...",
    "action": "TICKET_FILED",
    "proposedTicket": {
      "title": "Okta account unlock — emp-4472",
      "assigneeGroup": "access-team",
      "priority": "P2",
      "body": "Employee locked out after self-service token expiry. Unlock and trigger new password reset flow."
    },
    "specialistTag": "access",
    "resolvedAt": "2026-06-28T09:15:22Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Category right; reason names the identity provider and the lockout signal.",
    "scoredAt": "2026-06-28T09:15:14Z"
  },
  "escalationReason": null,
  "status": "RESOLVED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:23Z"
}
```

A blocked request has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "TICKET_BLOCKED"`, `finishedAt = null`. An escalated request has `classification.category = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### ClassificationDecision (returned by `ClassifierAgent`, embedded in `Request.classification`)

```json
{ "category": "ACCESS" | "INFRASTRUCTURE" | "SOFTWARE" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### ProposedTicket (nested in `Resolution.proposedTicket`, optional)

```json
{
  "title": "Okta account unlock — emp-4472",
  "assigneeGroup": "access-team" | "infra-team" | "software-team",
  "priority": "P1" | "P2" | "P3",
  "body": "Full ticket description with approver if P1."
}
```

### Resolution (returned by any specialist, embedded in `Request.resolution`)

```json
{
  "responseSubject": "Re: …",
  "responseBody": "2–4 short paragraphs",
  "action": "TICKET_FILED" | "SELF_SERVICE_RESOLVED" | "RUNBOOK_LINKED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "proposedTicket": { … } | null,
  "specialistTag": "access" | "infra" | "software",
  "resolvedAt": "ISO-8601"
}
```

### GuardrailVerdict (returned by `TicketGuardrail`, embedded in `Request.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["open-ended-admin-access", "p1-without-approver", …],
  "rubricVersion": "v1" }
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
- `409` — `unblock` requested on a request not currently in `TICKET_BLOCKED`.
- `503` — backend unavailable (workflow timed out; recovery in progress).
