# API contract — meeting-facilitator

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `GET` | `/api/sessions` | — | `200 [ MeetingSession... ]` (sorted newest-first) | `MeetingEndpoint` ← `MeetingSessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 MeetingSession` / `404` | `MeetingEndpoint` ← `MeetingSessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `MeetingEndpoint` ← `MeetingSessionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### MeetingSession

```json
{
  "sessionId": "sess-8a3f…",
  "rawSegment": {
    "segmentId": "seg-001",
    "sessionId": "sess-8a3f…",
    "speakerRaw": "[REDACTED-NAME]",
    "textRaw": "...",
    "chatThreadRaw": "...",
    "capturedAt": "2026-06-28T09:00:00Z"
  },
  "sanitized": {
    "redactedTranscript": "The team reviewed the sprint velocity. A team member raised concern about [REDACTED-PROJECT] timeline.",
    "redactedChat": "One participant shared a link. Another asked for clarification on scope.",
    "piiCategoriesFound": ["name", "project-codename"]
  },
  "notes": {
    "title": "Sprint velocity review",
    "keyPoints": [
      "Current sprint velocity is below target.",
      "Timeline risk identified for an upcoming milestone.",
      "Testing coverage gaps were discussed."
    ],
    "actionItems": [
      "Engineering team to investigate velocity drop by end of week.",
      "Product team to revise milestone date estimate."
    ],
    "generatedAt": "2026-06-28T09:00:07Z"
  },
  "chatRecap": {
    "summary": "A resource link was shared in chat. One participant raised a scope clarification that was not addressed verbally.",
    "messageCount": 14,
    "generatedAt": "2026-06-28T09:00:09Z"
  },
  "guardrailVerdict": {
    "passed": true,
    "reason": "No PII tokens or policy violations detected."
  },
  "evalScore": 4,
  "evalRationale": "Notes are complete and anonymised; one action item lacks a concrete deadline.",
  "status": "PUBLISHED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:10Z"
}
```

### SSE event format

```
event: session-update
data: { "sessionId": "...", "status": "PUBLISHED", ... }
```

One event per state transition. Clients reconcile by `sessionId`.
