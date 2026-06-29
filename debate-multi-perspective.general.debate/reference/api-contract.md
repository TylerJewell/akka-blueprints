# API contract

All endpoints are JSON unless noted. ACL open to the internet (local-dev only). Base path `/api`.

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/debates` | `{ "topic": "string" }` | `{ "debateId": "uuid" }` | `DebateEndpoint` → `InboundRequestQueue` |
| GET | `/api/debates?status=...` | — | `{ "debates": [Debate, ...] }` | `DebateEndpoint` ← `DebatesView` (status filtered client-side) |
| GET | `/api/debates/{id}` | — | `Debate` or 404 | `DebateEndpoint` ← `DebatesView` |
| GET | `/api/debates/sse` | — | SSE stream of `Debate` | `DebateEndpoint` ← `DebatesView` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `DebateEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `DebateEndpoint` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `DebateEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`Debate` (lifecycle fields are nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "topic": "string or null",
  "status": "PENDING | DEBATING | SYNTHESIZING | CONCLUDED | FAILED",
  "currentRound": 0,
  "maxRounds": 5,
  "rounds": [
    { "number": 1, "advocateArgument": "string", "criticArgument": "string" }
  ],
  "conclusion": "string or null",
  "keyArguments": ["string", "..."],
  "qualityScore": 0.0,
  "createdAt": "ISO-8601 or null",
  "concludedAt": "ISO-8601 or null",
  "failedReason": "string or null"
}
```

`keyArguments`, `qualityScore`, `conclusion`, `concludedAt`, `failedReason` are `null` until their event fires.

## SSE event format

`GET /api/debates/sse` emits one `data:` line per `Debate` update, serialized as the JSON above:

```
data: {"id":"...","status":"DEBATING","currentRound":2,"rounds":[...], ...}

data: {"id":"...","status":"CONCLUDED","conclusion":"...","keyArguments":[...],"qualityScore":0.86, ...}
```

The UI subscribes on load and updates each debate row in place by `id`.
