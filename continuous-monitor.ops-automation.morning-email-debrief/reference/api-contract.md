# API contract — morning-email-debrief

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/debrief` | — | `200 [ DebriefRun... ]` (sorted newest-first). Optional `?status=READY\|ASSEMBLING\|CREATED\|FAILED`. | `DebriefEndpoint` ← `DebriefView` |
| `GET` | `/api/debrief/{runId}` | — | `200 DebriefRun` / `404` | `DebriefEndpoint` ← `DebriefView` |
| `GET` | `/api/debrief/{runId}/emails` | — | `200 [ EmailRecord... ]` for the run | `DebriefEndpoint` ← `EmailView` |
| `GET` | `/api/email/{emailId}` | — | `200 EmailRecord` / `404` | `DebriefEndpoint` ← `EmailView` |
| `GET` | `/api/debrief/sse` | — | `text/event-stream` | `DebriefEndpoint` ← `DebriefView` + `EmailView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### DebriefRun

```json
{
  "runId": "run-2026-06-28-0700",
  "totalEmails": 5,
  "summarisedCount": 5,
  "debrief": {
    "runId": "run-2026-06-28-0700",
    "narrativeSummary": "5 emails processed this morning. Two items require immediate attention: a pending legal review and an ops alert for elevated resource usage.",
    "entries": [
      { "emailId": "em-001", "oneLinerSummary": "Quarterly contract amendment review required before Friday.", "priority": "HIGH", "category": "legal" },
      { "emailId": "em-002", "oneLinerSummary": "CPU alert on a node at 94%; auto-scaling triggered, ticket opened.", "priority": "HIGH", "category": "ops" },
      { "emailId": "em-003", "oneLinerSummary": "Sprint 14 retrospective notes shared by team lead.", "priority": "MEDIUM", "category": "project" },
      { "emailId": "em-004", "oneLinerSummary": "Weekly KPI digest — no action required.", "priority": "LOW", "category": "automated" },
      { "emailId": "em-005", "oneLinerSummary": "Vendor invoice for Q2 services attached for approval.", "priority": "MEDIUM", "category": "billing" }
    ],
    "totalEmailCount": 5,
    "assembledAt": "2026-06-28T07:02:30Z"
  },
  "evalScore": 4,
  "evalRationale": "All HIGH-priority items are surfaced; tone is plain and appropriate.",
  "status": "READY",
  "createdAt": "2026-06-28T07:01:00Z",
  "readyAt": "2026-06-28T07:02:30Z"
}
```

### EmailRecord

```json
{
  "emailId": "em-001",
  "runId": "run-2026-06-28-0700",
  "incoming": {
    "emailId": "em-001",
    "runId": "run-2026-06-28-0700",
    "from": "[REDACTED-EMAIL]",
    "subject": "[REDACTED] — Quarterly legal review due",
    "body": "...",
    "receivedAt": "2026-06-28T07:00:05Z"
  },
  "sanitized": {
    "redactedSubject": "[REDACTED] — Quarterly legal review due",
    "redactedBody": "Please review the attached contract amendments before Friday.",
    "piiCategoriesFound": ["email", "name"]
  },
  "entry": {
    "emailId": "em-001",
    "oneLinerSummary": "Quarterly contract amendment review required before Friday.",
    "priority": "HIGH",
    "category": "legal"
  },
  "status": "SUMMARISED",
  "createdAt": "2026-06-28T07:00:05Z"
}
```

### SSE event format

```
event: debrief-update
data: { "runId": "run-2026-06-28-0700", "status": "READY", "summarisedCount": 5, ... }

event: email-update
data: { "emailId": "em-001", "runId": "run-2026-06-28-0700", "status": "SUMMARISED", ... }
```

One event per state transition. Clients reconcile debrief updates by `runId` and email updates by `emailId`.
