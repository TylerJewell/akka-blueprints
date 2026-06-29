# API contract — employee-helpdesk-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/questions` | — | `200 [ Question... ]` (newest-first); supports optional `?topic=…&status=…` filtered client-side | `HelpdeskEndpoint` ← `QuestionView` |
| `GET` | `/api/questions/{id}` | — | `200 Question` / `404` | `HelpdeskEndpoint` ← `QuestionView` |
| `POST` | `/api/questions` | `{ "employeeId": String, "channel": String, "subject": String, "body": String }` | `201 { "questionId": String }` | `HelpdeskEndpoint` → `QuestionQueue` |
| `POST` | `/api/questions/{id}/unblock` | `{ "decidedBy": String, "note": String }` | `200 Question` (status now `ANSWERED`) / `409` if not `BLOCKED` | `HelpdeskEndpoint` → `QuestionEntity` |
| `GET` | `/api/questions/sse` | — | `text/event-stream` | `HelpdeskEndpoint` ← `QuestionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### IncomingQuestion (request body for `POST /api/questions`, minus `questionId` and `receivedAt`)

```json
{
  "employeeId": "EMP-04821",
  "channel": "portal",
  "subject": "How do I request parental leave?",
  "body": "Hi, I need to understand the process for filing parental leave starting next quarter. My manager is EMP-04820. Contact: alice@example.com."
}
```

### Question

```json
{
  "questionId": "q-7e3a21f0",
  "incoming": {
    "questionId": "q-7e3a21f0",
    "employeeId": "EMP-04821",
    "channel": "portal",
    "subject": "How do I request parental leave?",
    "body": "...raw with PII...",
    "receivedAt": "2026-06-28T09:10:00Z"
  },
  "sanitized": {
    "redactedSubject": "How do I request parental leave?",
    "redactedBody": "Hi, I need to understand the process for filing parental leave starting next quarter. My manager is [REDACTED-EMP-ID]. Contact: [REDACTED-EMAIL].",
    "piiCategoriesFound": ["employee-id", "email"]
  },
  "routing": {
    "topic": "HR",
    "confidence": "high",
    "reason": "Direct leave-process question."
  },
  "answer": {
    "responseSubject": "Re: How do I request parental leave?",
    "responseBody": "Parental leave is requested through the HR portal under the Leave Management section...",
    "action": "INFO_PROVIDED",
    "specialistTag": "hr",
    "answeredAt": "2026-06-28T09:10:14Z"
  },
  "guardrail": {
    "allowed": true,
    "violations": [],
    "rubricVersion": "v1"
  },
  "routeScore": {
    "score": 5,
    "rationale": "Topic right; reason identifies the leave-process signal directly.",
    "scoredAt": "2026-06-28T09:10:08Z"
  },
  "escalationReason": null,
  "status": "ANSWERED",
  "createdAt": "2026-06-28T09:10:00Z",
  "finishedAt": "2026-06-28T09:10:15Z"
}
```

A blocked question has `guardrail.allowed = false`, `guardrail.violations` non-empty, `status = "BLOCKED"`, `finishedAt = null`. An escalated question has `routing.topic = "UNCLEAR"` (or a workflow error), `escalationReason` populated, `status = "ESCALATED"`.

### RoutingDecision (returned by `TopicRouter`, embedded in `Question.routing`)

```json
{ "topic": "HR" | "IT" | "POLICY" | "UNCLEAR",
  "confidence": "high" | "medium" | "low",
  "reason": "one short sentence" }
```

### Answer (returned by any specialist, embedded in `Question.answer`)

```json
{ "responseSubject": "Re: …",
  "responseBody": "3–5 short paragraphs",
  "action": "INFO_PROVIDED" | "POLICY_CITED" | "TICKET_OPENED" | "FOLLOW_UP_SCHEDULED" | "ESCALATED",
  "specialistTag": "hr" | "it" | "policy",
  "answeredAt": "ISO-8601" }
```

### GuardrailVerdict (returned by `AnswerGuardrail`, embedded in `Question.guardrail`)

```json
{ "allowed": true | false,
  "violations": ["invented-policy-reference", "echoes-redacted-token", …],
  "rubricVersion": "v1" }
```

### RouteScore (returned by `RouteJudge`, embedded in `Question.routeScore`)

```json
{ "score": 1..5,
  "rationale": "one short sentence",
  "scoredAt": "ISO-8601" }
```

## SSE event format

```
event: question-update
data: { "questionId": "q-…", "status": "ROUTED", … full Question JSON … }
```

One event per state transition on the `QuestionView`. Clients reconcile by `questionId`.

## Status codes

- `200` — successful read or successful state transition.
- `201` — successful create (`POST /api/questions`).
- `404` — unknown `questionId`.
- `409` — `unblock` requested on a question not currently in `BLOCKED`.
- `503` — backend unavailable (workflow timed out, recovery in progress).
