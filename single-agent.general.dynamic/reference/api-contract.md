# API contract — Dynamic Route Agent

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/requests` | `SubmitRequest` | `{ "requestId": "uuid" }` | RouteEndpoint |
| GET | `/api/requests` | — | `{ "requests": [RequestRecord, ...] }` | RouteEndpoint |
| GET | `/api/requests/{id}` | — | `RequestRecord` or 404 | RouteEndpoint |
| GET | `/api/requests/sse` | — | Server-Sent Events of `RequestRecord` | RouteEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | RouteEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | RouteEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | RouteEndpoint |
| GET | `/` | — | 302 -> `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static resource | AppEndpoint |

## Payload schemas

`SubmitRequest` (inbound JSON):

```json
{
  "route": "SUMMARIZE | TRANSLATE",
  "text": "string",
  "targetLanguage": "string (required when route is TRANSLATE, else omitted)"
}
```

`RequestRecord` (entity state and view row; lifecycle fields are nullable, declared `Optional<T>` in Java, serialized as the raw value or `null`):

```json
{
  "id": "uuid",
  "route": "SUMMARIZE | TRANSLATE",
  "inputText": "string",
  "status": "SUBMITTED | COMPLETED | BLOCKED",
  "submittedAt": "ISO-8601",
  "targetLanguage": "string or null",
  "output": "string or null",
  "completedAt": "ISO-8601 or null",
  "blockedReason": "string or null",
  "blockedAt": "ISO-8601 or null"
}
```

`POST /api/requests` response:

```json
{ "requestId": "uuid" }
```

## SSE event format

`GET /api/requests/sse` streams each `RequestRecord` as it changes:

```
event: request
data: {"id":"...","route":"SUMMARIZE","status":"SUBMITTED",...}

event: request
data: {"id":"...","route":"SUMMARIZE","status":"COMPLETED","output":"...",...}
```

The stream is driven by `RequestsView` updates from `RequestEntity` events. Clients render or replace the card keyed by `id`.
