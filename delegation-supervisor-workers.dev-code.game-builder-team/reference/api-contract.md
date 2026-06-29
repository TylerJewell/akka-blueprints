# API contract — game-builder-team

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Game endpoints (`GameEndpoint`)

### `POST /api/games`
Submit a game idea. Enqueues to `BuildRequestQueue`.

Request:
```json
{ "idea": "a snake game where the snake speeds up over time" }
```
Response `200`:
```json
{ "projectId": "uuid" }
```

### `GET /api/games?status=DELIVERED`
List projects. The optional `status` query filters client-side (the view exposes only `getAllProjects`; Akka cannot auto-index the enum column).

Response `200`:
```json
{ "projects": [ /* GameProject */ ] }
```

### `GET /api/games/{projectId}`
Single project. `200` with a `GameProject`, or `404`.

### `GET /api/games/sse`
Server-Sent Events stream of `GameProject` updates. Content type `text/event-stream`. Each event:
```
event: project
data: { /* GameProject JSON */ }
```

## Metadata endpoints (`GameEndpoint`)

```
GET /api/metadata/eval-matrix  -> text/yaml      (src/main/resources/metadata/eval-matrix.yaml)
GET /api/metadata/risk-survey  -> text/yaml      (src/main/resources/metadata/risk-survey.yaml)
GET /api/metadata/readme       -> text/markdown  (src/main/resources/metadata/README.md)
```

## App endpoints (`AppEndpoint`)

```
GET /              -> 302 /app/index.html
GET /app/{*path}   -> static-resources/{*path}
```

## Payload shapes

`GameProject` (lifecycle fields are nullable — `Optional` in Java, raw value or `null` on the wire):
```json
{
  "id": "uuid",
  "idea": "string",
  "status": "QUEUED | DESIGNING | CODING | TESTING | TEST_FAILED | DELIVERED | BLOCKED",
  "queuedAt": "ISO-8601 or null",
  "designedAt": "ISO-8601 or null",
  "spec": { "title": "string", "genre": "string", "mechanics": ["string"], "controls": "string", "winCondition": "string" },
  "codedAt": "ISO-8601 or null",
  "code": { "html": "string", "entryPoint": "string", "filesTouched": ["string"] },
  "testedAt": "ISO-8601 or null",
  "testReport": { "passed": true, "checks": ["string"], "failures": ["string"] },
  "testAttempts": 0,
  "deliveredAt": "ISO-8601 or null",
  "blockReason": "string or null"
}
```

`spec`, `code`, `testReport`, and the timestamp fields are `null` until their transition fires.
