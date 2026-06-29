# API contract — doc-linter

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only).

## Endpoints

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/lint` | `{ "filePath": "string" }` | `{ "runId": "uuid" }` | LintEndpoint → LinterAgent, LintEntity |
| POST | `/api/lint/{id}/feedback` | `{ "rule": "string", "note": "string" }` | `200` \| `404` | LintEndpoint → LintEntity |
| GET | `/api/lint` | — | `{ "runs": [LintRun, ...] }` | LintEndpoint ← LintView |
| GET | `/api/lint/{id}` | — | `LintRun` \| `404` | LintEndpoint ← LintView |
| GET | `/api/lint/sse` | — | SSE of `LintRun` | LintEndpoint ← LintView |
| GET | `/api/files` | — | `{ "files": ["path", ...] }` | LintEndpoint |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | LintEndpoint |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | LintEndpoint |
| GET | `/api/metadata/readme` | — | `text/markdown` | LintEndpoint |
| GET | `/` | — | `302 → /app/index.html` | AppEndpoint |
| GET | `/app/{*path}` | — | static file | AppEndpoint |

## Payload shapes

`LintRun` (lifecycle fields nullable — `Optional<T>` in Java, raw value or `null` on the wire):

```json
{
  "id": "uuid",
  "filePath": "sample-data/getting-started.md",
  "status": "REQUESTED | LINTED | FEEDBACK_APPLIED | BLOCKED",
  "requestedAt": "ISO-8601",
  "lintedAt": "ISO-8601 or null",
  "findings": [
    { "rule": "no-h1", "line": 1, "severity": "ERROR | WARNING | INFO", "message": "string" }
  ],
  "summary": "string or null",
  "gate": "PASS | FAIL or null",
  "feedbackNote": "string or null",
  "feedbackAt": "ISO-8601 or null",
  "blockReason": "string or null"
}
```

`LintResult` (agent return, internal):

```json
{
  "orderedFindingRules": ["no-h1", "line-length"],
  "summary": "string"
}
```

## SSE event format

`GET /api/lint/sse` streams the `LintView` projection. Each event is a single `LintRun`
JSON object as `data:`, emitted whenever a run is created or changes state. The UI keys on
`id` to upsert rows in the live list.
