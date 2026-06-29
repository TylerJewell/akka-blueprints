# API contract — traced-agent-otel

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/conversations` | `SubmitConversationRequest` | `201 { conversationId }` | `ConversationEndpoint` → `ConversationEntity` + `TraceExportWorkflow` |
| `GET` | `/api/conversations` | — | `200 [ Conversation... ]` (newest-first) | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/{id}` | — | `200 Conversation` / `404` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/conversations/sse` | — | `text/event-stream` | `ConversationEndpoint` ← `ConversationView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitConversationRequest (request body)

```json
{
  "promptText": "Write a Java method that parses an ISO-8601 date string and returns a LocalDate.",
  "toolModeEnabled": false,
  "submittedBy": "dev-user-01"
}
```

### Conversation (response body)

```json
{
  "conversationId": "c-7a3f...",
  "request": {
    "conversationId": "c-7a3f...",
    "promptText": "Write a Java method that parses an ISO-8601 date string and returns a LocalDate.",
    "toolModeEnabled": false,
    "submittedBy": "dev-user-01",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "agentResponse": {
    "answerText": "Here is a Java method to parse ISO-8601 dates: ...",
    "spans": [
      {
        "spanId": "a1b2c3d4",
        "parentSpanId": null,
        "traceId": "trace-7f3e9a",
        "operationName": "llm.completion",
        "kind": "LLM_CALL",
        "startTime": "2026-06-28T10:00:01.000Z",
        "endTime": "2026-06-28T10:00:03.142Z",
        "attributes": { "model": "claude-sonnet-4-6", "token.count": "218" },
        "status": "OK",
        "errorMessage": null
      }
    ],
    "respondedAt": "2026-06-28T10:00:03.142Z"
  },
  "spanSummary": {
    "traceId": "trace-7f3e9a",
    "totalSpans": 1,
    "llmTurnCount": 1,
    "toolCallCount": 0,
    "errorCount": 0,
    "wallClockMillis": 2142,
    "spans": [ "... same span list as agentResponse.spans ..." ],
    "collectedAt": "2026-06-28T10:00:03.500Z"
  },
  "flushResult": {
    "spansExported": 1,
    "spansDropped": 0,
    "exporterHealthy": true,
    "zipkinTraceUrl": "http://localhost:9411/zipkin/traces/trace-7f3e9a",
    "flushedAt": "2026-06-28T10:00:03.700Z"
  },
  "monitorResult": {
    "qualityScore": 5,
    "p50Millis": 2142,
    "p95Millis": 2142,
    "toolSuccessRate": 1.0,
    "rationale": "All LLM spans have non-zero duration; tool success rate N/A (no tool calls); span count matches expected.",
    "monitoredAt": "2026-06-28T10:00:03.900Z"
  },
  "status": "MONITORED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:03.900Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### ConversationRow (view row — returned by list and SSE)

The view row mirrors `Conversation` but omits `agentResponse.spans` raw list. The full span list is available only on the single-item endpoint `GET /api/conversations/{id}`. The `spanSummary` aggregates (counts, wallClockMillis) are included in the row.

### SSE event format

```
event: conversation-update
data: { "conversationId": "c-7a3f...", "status": "MONITORED", "monitorResult": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `COMPLETED`, `FLUSHED`, `MONITORED`, `FAILED`). Each event carries the full row at the moment of transition; a late-joining client never needs to replay. Clients reconcile by `conversationId`.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `ConversationEndpoint` with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
