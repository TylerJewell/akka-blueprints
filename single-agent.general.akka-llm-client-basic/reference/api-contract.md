# API contract — akka-llm-client-basic

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `{}` | `201 { sessionId }` | `ConversationEndpoint` → `ConversationEntity` |
| `POST` | `/api/conversations/{sessionId}/turns` | `TurnRequest` | `200 { turnId }` | `ConversationEndpoint` → `ConversationAgent` → `ConversationEntity` |
| `GET` | `/api/conversations` | — | `200 [ ConversationSession... ]` (newest-first) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{sessionId}` | — | `200 ConversationSession` / `404` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{sessionId}/sse` | — | `text/event-stream` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `ConversationEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `ConversationEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `ConversationEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### TurnRequest (request body for POST /turns)

```json
{
  "prompt": "What is the capital of Iceland?",
  "modelOverride": null
}
```

`modelOverride` is optional. When present, it must be one of the provider keys configured in `application.conf`. When absent, the default configured provider is used.

The endpoint returns `400 Bad Request` if `prompt` is blank or missing — no agent call is made.

### ConversationSession (response body)

```json
{
  "sessionId": "s-3a7f...",
  "turns": [
    {
      "turnId": "t-1b2c...",
      "prompt": "What is the capital of Iceland?",
      "reply": {
        "text": "Reykjavik. It is also the country's largest city, with a population of roughly 130 000 in the city proper.",
        "inputTokens": 14,
        "outputTokens": 22,
        "latencyMs": 1340,
        "generatedAt": "2026-06-28T10:00:01Z"
      },
      "status": "REPLIED",
      "submittedAt": "2026-06-28T10:00:00Z",
      "repliedAt": "2026-06-28T10:00:01Z"
    }
  ],
  "status": "ACTIVE",
  "createdAt": "2026-06-28T09:59:58Z",
  "lastActivityAt": "2026-06-28T10:00:01Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). A `PENDING` turn has `reply = null` and `repliedAt = null`.

### POST /api/conversations response

```json
{ "sessionId": "s-3a7f..." }
```

### POST /api/conversations/{sessionId}/turns response

```json
{ "turnId": "t-1b2c..." }
```

The turn's reply is not returned inline — the caller reads the full session via `GET /api/conversations/{sessionId}` or receives it through the SSE stream.

### SSE event format

```
event: turn-update
data: { "sessionId": "s-3a7f...", "turnId": "t-1b2c...", "status": "REPLIED", "reply": { ... }, ... }
```

One event per turn-status change (`PENDING`, `REPLIED`, `FAILED`) and one on `SessionCreated`. Clients reconcile by `sessionId` + `turnId`; an event always carries the full turn at the moment of transition, so a late-joining client never needs to replay.

## Error responses

| Status | Condition |
|---|---|
| `400` | `prompt` is blank or missing in POST /turns. |
| `404` | `sessionId` not found in GET or SSE. |
| `422` | Agent exhausted all 3 guardrail-retry iterations; turn recorded as FAILED. |
| `504` | Agent call exceeded 30 s server-side timeout; turn recorded as FAILED. |

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware. `submittedBy` tracking is not present in this baseline — it is introduced in subsequent quickstart blueprints that add per-user session isolation.
