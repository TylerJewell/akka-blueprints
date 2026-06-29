# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/translation` | `{ "sourceText": "string", "targetLanguage": "string" }` | `{ "jobId": "uuid" }` | `TranslationEndpoint` → `JobQueue` |
| GET | `/api/translation` | — | `{ "jobs": [TranslationJobRow, ...] }` | `TranslationEndpoint` → `TranslationView` |
| GET | `/api/translation?status=SELECTED` | — | filtered list (client-side filter) | `TranslationEndpoint` |
| GET | `/api/translation/{id}` | — | `TranslationJobRow` or 404 | `TranslationEndpoint` |
| GET | `/api/translation/sse` | — | `text/event-stream` of `TranslationJobRow` | `TranslationEndpoint` → `TranslationView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `TranslationEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `TranslationEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `TranslationEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/translation` request:

```json
{ "sourceText": "The server encountered an unexpected condition.", "targetLanguage": "Spanish" }
```

`TranslationJobRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "jobId": "uuid",
  "sourceText": "string",
  "targetLanguage": "string",
  "status": "PENDING | IN_PROGRESS | SELECTED | PARTIAL | BLOCKED",
  "variants": [
    { "register": "formal", "translatedText": "...", "producedAt": "ISO-8601" },
    { "register": "informal", "translatedText": "...", "producedAt": "ISO-8601" },
    { "register": "literal", "translatedText": "...", "producedAt": "ISO-8601" }
  ],
  "selection": {
    "selectedVariant": { "register": "formal", "translatedText": "...", "producedAt": "ISO-8601" },
    "selectionRationale": "string",
    "guardrailVerdict": "ok",
    "selectedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/translation/sse` emits one event per job change:

```
event: job
data: { ...TranslationJobRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `jobId`. No polling.
