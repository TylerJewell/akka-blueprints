# API contract — meeting-to-tasks

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Served by `MeetingEndpoint` (`/api/*`) and `AppEndpoint` (`/`, `/app/*`).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/meetings` | `{ "title": string, "rawNotes": string }` | `{ "meetingId": string }` | MeetingEndpoint → MeetingPipelineWorkflow |
| POST | `/api/meetings/{id}/approve` | `{ "approvedBy": string }` | `200` \| `404` | MeetingEndpoint → MeetingEntity |
| POST | `/api/meetings/{id}/reject` | `{ "reason": string }` | `200` \| `404` | MeetingEndpoint → MeetingEntity |
| GET | `/api/meetings?status=...` | — | `{ "meetings": [Meeting, ...] }` | MeetingEndpoint → MeetingsView (client-side filter) |
| GET | `/api/meetings/{id}` | — | `Meeting` \| `404` | MeetingEndpoint → MeetingsView |
| GET | `/api/meetings/sse` | — | SSE of `Meeting` | MeetingEndpoint → MeetingsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | MeetingEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | MeetingEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | MeetingEndpoint |
| GET | `/` | — | `302 → /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `?status=` filter is applied in the endpoint over `getAllMeetings`; the view has no enum WHERE clause (Lesson 2). `GET` with `HEAD` (`curl -I`) returns 404 on these `@Get` routes — probe with a bare GET (Lesson 15).

## Meeting JSON form

Lifecycle fields are `Optional` in Java; on the wire they are the raw value or `null` (Akka serializes `Optional<T>` transparently — Lesson 6).

```json
{
  "id": "uuid",
  "title": "string",
  "rawNotes": "string",
  "status": "RECEIVED | SANITIZED | EXTRACTED | AWAITING_APPROVAL | APPROVED | REJECTED | COMPLETED | FAILED",
  "receivedAt": "ISO-8601",
  "sanitizedNotes": "string or null",
  "redactionCount": "int or null",
  "extractedTasks": "[TaskItem, ...] or null",
  "approvedAt": "ISO-8601 or null",
  "approvedBy": "string or null",
  "rejectedAt": "ISO-8601 or null",
  "rejectReason": "string or null",
  "trelloBoardUrl": "string or null",
  "trelloCardCount": "int or null",
  "csvPath": "string or null",
  "slackMessageTs": "string or null",
  "completedAt": "ISO-8601 or null",
  "failedAt": "ISO-8601 or null",
  "failureReason": "string or null"
}
```

`TaskItem`:

```json
{ "title": "string", "assignee": "string", "dueDate": "YYYY-MM-DD or empty", "priority": "low | medium | high" }
```

## SSE event format

`GET /api/meetings/sse` streams via `serverSentEventsForView` over `MeetingsView`. Each event:

```
event: meeting
data: { ...Meeting JSON as above... }
```

The UI subscribes with `EventSource`, upserts each meeting into the live list by `id`, and re-renders the per-meeting controls when `status` is `AWAITING_APPROVAL`.
