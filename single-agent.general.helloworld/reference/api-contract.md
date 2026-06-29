# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/ask` | `{ "question": "string" }` | `{ "questionId": "uuid" }` | `AskEndpoint` |
| GET | `/api/questions` | — (optional `?status=ASKED\|ANSWERED\|BLOCKED`) | `{ "questions": [Question, ...] }` | `AskEndpoint` |
| GET | `/api/questions/{id}` | — | `Question` \| 404 | `AskEndpoint` |
| GET | `/api/questions/sse` | — | `text/event-stream` of `Question` | `AskEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `AskEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `AskEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `AskEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static-resources file | `AppEndpoint` |

`GET /api/questions?status=...` filters client-side from `getAllQuestions` because the status enum column cannot be auto-indexed (Lesson 2).

## Payload shapes

`Question` JSON (lifecycle fields are nullable — `Optional` in Java, serialized as the raw value or `null`):

```json
{
  "id": "uuid",
  "text": "string",
  "status": "ASKED | ANSWERED | BLOCKED",
  "askedAt": "ISO-8601 or null",
  "answerText": "string or null",
  "confidence": 0.0,
  "answeredAt": "ISO-8601 or null",
  "blockedReason": "string or null",
  "blockedAt": "ISO-8601 or null"
}
```

`confidence` is `null` until an answer is recorded.

## SSE event format

`GET /api/questions/sse` streams each `Question` row as it changes:

```
event: question
data: { ...Question JSON as above... }
```

The stream is backed by `componentClient.forView().method(QuestionsView::streamAllQuestions).source()` mapped to SSE events; the UI's App tab consumes it with `EventSource` and upserts rows by `id`.
