# API contract

All `/api` endpoints are JSON. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/meetings` | `{ title, notes }` | `{ meetingId }` | `MeetingEndpoint` |
| GET | `/api/meetings?status=...` | — | `{ meetings: [Meeting] }` | `MeetingEndpoint` |
| GET | `/api/meetings/{id}` | — | `Meeting` | `MeetingEndpoint` |
| GET | `/api/meetings/sse` | — | SSE of `Meeting` | `MeetingEndpoint` |
| POST | `/sim/trello/cards` | `{ name, idempotencyKey }` | `{ cardUrl }` | `SimEndpoint` |
| POST | `/sim/slack/messages` | `{ channel, text, idempotencyKey }` | `{ ts }` | `SimEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `MeetingEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `MeetingEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `MeetingEndpoint` |
| GET | `/` | — | `302 /app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static file | `AppEndpoint` |

The `?status=` filter on the list endpoint is applied client-side after `getAllMeetings` (the view cannot index the enum column, Lesson 2).

## Payload shapes

`Meeting` (lifecycle fields are nullable — `Optional` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "title": "string or null",
  "status": "RECEIVED | SANITIZED | EXTRACTED | DISPATCHED | FAILED",
  "receivedAt": "ISO-8601 or null",
  "sanitizedNotes": "string or null",
  "redactionCount": "int or null",
  "sanitizedAt": "ISO-8601 or null",
  "summary": "string or null",
  "actions": [
    { "title": "string", "assignee": "string or null", "dueHint": "string or null", "trelloCardUrl": "string or null" }
  ],
  "extractedAt": "ISO-8601 or null",
  "dispatchedAt": "ISO-8601 or null",
  "trelloBoardId": "string or null",
  "slackChannel": "string or null",
  "slackMessageTs": "string or null",
  "failureReason": "string or null",
  "failedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/meetings/sse` streams `MeetingsView` updates. Each event:

```
event: meeting
data: { ...Meeting JSON as above... }
```

The browser subscribes with `EventSource` and upserts each meeting into the list by `id`.
