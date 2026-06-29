# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/email-threads` | `{ "sender": "string", "subject": "string", "body": "string" }` | `{ "threadId": "uuid" }` | EmailEndpoint → EmailWorkflow |
| POST | `/api/threads/{threadId}/approve` | `{ "approvedBy": "string", "note": "string" }` | `200` \| `404` | EmailEndpoint → EmailThreadEntity |
| POST | `/api/threads/{threadId}/dismiss` | `{ "dismissedBy": "string", "note": "string" }` | `200` \| `404` | EmailEndpoint → EmailThreadEntity |
| GET | `/api/threads` | — | `{ "threads": [EmailThread, ...] }` | EmailEndpoint → ThreadsView |
| GET | `/api/threads/{threadId}` | — | `EmailThread` \| `404` | EmailEndpoint → EmailThreadEntity |
| GET | `/api/threads/sse` | — | SSE stream of `EmailThread` | EmailEndpoint → ThreadsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | EmailEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | EmailEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | EmailEndpoint |
| GET | `/` | — | `302 /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`EmailThread` (lifecycle fields are nullable; `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "sender": "string or null",
  "subject": "string or null",
  "status": "RECEIVED | TRIAGED | ACTION_DRAFTED | APPROVED | REPLY_SENT | MEETING_SCHEDULED | DISMISSED",
  "receivedAt": "ISO-8601 or null",
  "category": "string or null",
  "urgency": "string or null",
  "suggestedAction": "string or null",
  "triageScore": "number or null",
  "draftSubject": "string or null",
  "draftBody": "string or null",
  "proposedMeetingTitle": "string or null",
  "proposedMeetingTime": "ISO-8601 or null",
  "proposedAttendees": "string or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "approverNote": "string or null",
  "dismissedAt": "ISO-8601 or null",
  "dismissedBy": "string or null",
  "dismissNote": "string or null",
  "actionCompletedAt": "ISO-8601 or null",
  "actionReference": "string or null"
}
```

`EvalEvent` (emitted to the eval log topic; not exposed over REST):

```json
{
  "threadId": "uuid",
  "agentName": "string",
  "inputSummary": "string (sanitized)",
  "outputSummary": "string",
  "judgeScore": "number (0–1)",
  "evaluatedAt": "ISO-8601"
}
```

## SSE event format

`GET /api/threads/sse` streams the `ThreadsView` rows. Each message:

```
event: thread
data: { ...EmailThread... }
```

The UI subscribes once and replaces the matching row by `id` on each event.
