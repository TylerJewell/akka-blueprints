# API contract — screenplay-writer

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/threads` | `{ "sourceLabel": "string", "rawText": "string" }` | `{ "jobId": "uuid" }` | ThreadIntakeEndpoint |
| GET | `/api/jobs?status=...` | — | `{ "jobs": [ScreenplayJob, ...] }` | ThreadIntakeEndpoint |
| GET | `/api/jobs/{jobId}` | — | `ScreenplayJob` or 404 | ThreadIntakeEndpoint |
| GET | `/api/jobs/sse` | — | SSE stream of `ScreenplayJob` | ThreadIntakeEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | ThreadIntakeEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | ThreadIntakeEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | ThreadIntakeEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `status` query parameter is filtered client-side in the endpoint after calling `ScreenplayView.getAllJobs` — the view has no `WHERE status` query because Akka cannot auto-index enum columns (Lesson 2).

## ScreenplayJob JSON

Lifecycle fields are `Optional` in Java; on the wire they are the raw value or `null`.

```json
{
  "id": "uuid",
  "sourceLabel": "string",
  "status": "RECEIVED | SANITIZED | ANALYZED | DIALOGUE_WRITTEN | COMPLETED | BLOCKED | FAILED",
  "sanitizedText": "string or null",
  "redactionCount": "int or null",
  "characters": ["string", "..."],
  "setting": "string or null",
  "synopsis": "string or null",
  "tone": "string or null",
  "dialogue": "string or null",
  "screenplay": "string or null",
  "piiFindings": ["string", "..."],
  "sanitizedAt": "ISO-8601 or null",
  "analyzedAt": "ISO-8601 or null",
  "dialogueAt": "ISO-8601 or null",
  "completedAt": "ISO-8601 or null",
  "failureReason": "string or null"
}
```

`characters` and `piiFindings` are empty lists until their producing stage runs, not null.

## SSE event format

`GET /api/jobs/sse` streams one `data:` line per update, each a full `ScreenplayJob` JSON object as above. The UI keys on `id` and replaces the matching row on each event. No custom event names; the default `message` event carries the payload.
