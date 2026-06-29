# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/runs` | `{ "query": "string" }` | `{ "runId": "uuid" }` | `RunEndpoint` → `QueryQueue` |
| GET | `/api/runs` | — | `{ "runs": [AgentRunRow, ...] }` | `RunEndpoint` → `RunView` |
| GET | `/api/runs?status=TRACED` | — | filtered list (client-side filter) | `RunEndpoint` |
| GET | `/api/runs/{id}` | — | `AgentRunRow` or 404 | `RunEndpoint` |
| GET | `/api/runs/sse` | — | `text/event-stream` of `AgentRunRow` | `RunEndpoint` → `RunView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `RunEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `RunEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `RunEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/runs` request:

```json
{ "query": "What are the main open-source large language model evaluation frameworks?" }
```

`AgentRunRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "runId": "uuid",
  "query": "string",
  "status": "DISPATCHED | SEARCHING | TRACED | PARTIAL | BLOCKED_TOOL",
  "plan": {
    "searchQuery": "string",
    "maxPages": 5
  },
  "searchResults": {
    "results": [
      { "url": "https://...", "title": "string", "excerpt": "string", "tokensConsumed": 312 }
    ],
    "totalTokens": 1240,
    "durationMs": 4800
  },
  "trace": {
    "spans": [
      {
        "spanId": "uuid",
        "stepName": "searchStep",
        "toolName": "WebSearch",
        "inputTokens": 84,
        "outputTokens": 228,
        "durationMs": 1200,
        "startedAt": "ISO-8601"
      }
    ],
    "totalTokens": 1240,
    "totalDurationMs": 4800
  },
  "traceReport": {
    "summary": "string",
    "hotStep": "searchStep",
    "tokenBreakdown": { "searchStep": 980, "planStep": 260 }
  },
  "answer": {
    "answer": "string",
    "sources": ["https://..."],
    "guardrailVerdict": "ok",
    "synthesisedAt": "ISO-8601"
  },
  "blockedUrl": "string or null",
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "createdAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/runs/sse` emits one event per run change:

```
event: run
data: { ...AgentRunRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `runId`. No polling.
