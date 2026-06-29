# API contract — conditional-recruiting-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/applications` | — | `200 [ Application... ]` (newest-first); optional `?route=…&status=…` filtered client-side | `RecruitingEndpoint` ← `ApplicationView` |
| `GET` | `/api/applications/{id}` | — | `200 Application` / `404` | `RecruitingEndpoint` ← `ApplicationView` |
| `POST` | `/api/applications` | `{ "candidateId": String, "channel": String, "subject": String, "body": String }` | `201 { "applicationId": String }` | `RecruitingEndpoint` → `InboxQueue` |
| `POST` | `/api/applications/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Application` (status now `COMPLETED`) / `409` if not `TOOL_BLOCKED` | `RecruitingEndpoint` → `ApplicationEntity` |
| `GET` | `/api/applications/sse` | — | `text/event-stream` | `RecruitingEndpoint` ← `ApplicationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### InboundEmail (request body for `POST /api/applications`, minus `applicationId` and `receivedAt`)

```json
{
  "candidateId": "cand-4421",
  "channel": "email",
  "subject": "Available for the technical round",
  "body": "Hi, I can do Tuesday or Wednesday afternoon next week. Email me at [REDACTED-EMAIL]. Phone: [REDACTED-PHONE]."
}
```

### Application

```json
{
  "applicationId": "app-7cd3a91b",
  "incoming": {
    "applicationId": "app-7cd3a91b",
    "candidateId": "cand-4421",
    "channel": "email",
    "subject": "Available for the technical round",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T09:15:00Z"
  },
  "sanitized": {
    "redactedSubject": "Available for the technical round",
    "redactedBody": "Hi, I can do Tuesday or Wednesday afternoon next week. Email me at [REDACTED-EMAIL]. Phone: [REDACTED-PHONE].",
    "piiCategoriesFound": ["email", "phone"]
  },
  "routing": {
    "route": "INTERVIEW_REQUEST",
    "confidence": "high",
    "reason": "Explicit availability submission for a named interview round."
  },
  "reply": null,
  "calendar": {
    "interviewFormat": "technical",
    "interviewerId": "iv-17",
    "proposedSlot": "2026-07-01T14:00:00Z",
    "outcome": "SLOT_BOOKED",
    "specialistTag": "interview-organizer",
    "confirmedAt": "2026-06-28T09:15:22Z"
  },
  "toolGuardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routingScore": {
    "score": 5,
    "rationale": "Route correct; reason names the availability-submission signal directly.",
    "scoredAt": "2026-06-28T09:15:14Z"
  },
  "closureReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T09:15:00Z",
  "finishedAt": "2026-06-28T09:15:23Z"
}
```

A tool-blocked application has `toolGuardrail.allowed = false`, `toolGuardrail.violations` non-empty, `status = "TOOL_BLOCKED"`, `finishedAt = null`. An unroutable application has `routing.route = "UNROUTABLE"`, `closureReason` populated, `status = "UNROUTABLE_CLOSED"`.

### RoutingDecision (returned by `EmailClassifierAgent`, embedded in `Application.routing`)

```json
{ "route": "INFO_REQUEST" | "INTERVIEW_REQUEST" | "UNROUTABLE",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### OutboundReply (returned by `InfoRequester`, embedded in `Application.reply`)

```json
{ "replySubject": "Re: …",
  "replyBody": "2–4 short paragraphs",
  "action": "QUESTION_ANSWERED" | "ARTICLE_LINKED" | "FOLLOW_UP_PROMISED" | "ESCALATED",
  "specialistTag": "info-requester",
  "repliedAt": "ISO-8601" }
```

### CalendarConfirmation (returned by `InterviewOrganizer`, embedded in `Application.calendar`)

```json
{ "interviewFormat": "phone-screen" | "technical" | "panel" | "executive",
  "interviewerId": "iv-##",
  "proposedSlot": "ISO-8601",
  "outcome": "SLOT_BOOKED" | "CANDIDATE_DEFERRED" | "ESCALATED_TO_RECRUITER",
  "specialistTag": "interview-organizer",
  "confirmedAt": "ISO-8601" }
```

### ToolCallVerdict (returned by `ToolCallGuardrail`, embedded in `Application.toolGuardrail`)

```json
{ "allowed": true | false,
  "violations": ["past-dated-slot", "interviewer-id-not-found", …],
  "rubricVersion": "v1" }
```

### RoutingScore (returned by `RoutingJudge`, embedded in `Application.routingScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: application-update
data: { "applicationId": "app-…", "status": "CLASSIFIED", … full Application JSON … }
```

One event per state transition on the `ApplicationView`. Clients reconcile by `applicationId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/applications`).
- `404` — unknown `applicationId`.
- `409` — `unblock` requested on an application not currently in `TOOL_BLOCKED`.
- `503` — backend unavailable (workflow timed out, recovery in progress).
