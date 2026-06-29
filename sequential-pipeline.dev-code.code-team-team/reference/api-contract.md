# API contract — Game Builder Team

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Served by `BuildEndpoint` (`/api/*`) and `AppEndpoint` (`/`, `/app/*`).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/build-request` | `{ "brief": "string" }` | `{ "buildId": "uuid" }` | BuildEndpoint → InboundBriefQueue |
| GET | `/api/builds` | — | `{ "builds": [Build, ...] }` | BuildEndpoint → BuildsView |
| GET | `/api/builds/{buildId}` | — | `Build` or 404 | BuildEndpoint → BuildsView |
| GET | `/api/builds/sse` | — | SSE stream of `Build` | BuildEndpoint → BuildsView |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | BuildEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | BuildEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | BuildEndpoint |
| GET | `/` | — | 302 → `/app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

The `GET /api/builds` endpoint accepts an optional `?status=` query parameter and filters the `getAllBuilds` result client-side (the view has no indexable status column).

## Payload shapes

`build-request` body:

```json
{ "brief": "a number-guessing CLI game" }
```

`build-request` response:

```json
{ "buildId": "9f1c2e7a-..." }
```

`Build` JSON (lifecycle fields are nullable — `Optional<T>` in Java, serialized as the raw value or `null`):

```json
{
  "id": "uuid",
  "brief": "string or null",
  "status": "QUEUED | ENGINEERED | QA | SHIPPED | REJECTED",
  "engineeredAt": "ISO-8601 or null",
  "filename": "string or null",
  "sourceCode": "string or null",
  "qaAt": "ISO-8601 or null",
  "qaPassed": "boolean or null",
  "qaScore": "int or null",
  "qaNotes": "string or null",
  "chiefReviewedAt": "ISO-8601 or null",
  "shipped": "boolean or null",
  "shipSummary": "string or null",
  "reworkReason": "string or null"
}
```

## SSE event format

`GET /api/builds/sse` streams `Build` objects as they change, one per `data:` frame:

```
event: build
data: {"id":"9f1c...","status":"QA","filename":"guess.py","qaScore":92, ...}

```

Each frame carries the full `Build` JSON above. The App UI replaces the matching row by `id` on each frame.
