# API contract — akka-bridge

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `RunEndpoint` → `RunEntity` + `RunWorkflow` |
| `GET` | `/api/runs` | — | `200 [ Run... ]` (newest-first) | `RunEndpoint` ← `RunView` |
| `GET` | `/api/runs/{id}` | — | `200 Run` / `404` | `RunEndpoint` ← `RunView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `RunEndpoint` ← `RunView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "taskDescription": "Search the web for the top 3 causes of P99 latency in JVM-based services and summarise.",
  "permittedTools": [
    {
      "toolName": "web_search",
      "description": "Search for information on the web.",
      "argSchema": "{\"type\":\"object\",\"required\":[\"query\"],\"properties\":{\"query\":{\"type\":\"string\"}}}"
    },
    {
      "toolName": "summarize",
      "description": "Summarise a block of text into a concise paragraph.",
      "argSchema": "{\"type\":\"object\",\"required\":[\"text\"],\"properties\":{\"text\":{\"type\":\"string\"}}}"
    }
  ],
  "toolBudget": 4,
  "submittedBy": "operator-01"
}
```

### Run (response body)

```json
{
  "runId": "run-3ae...",
  "request": {
    "runId": "run-3ae...",
    "taskDescription": "Search the web for the top 3 causes of P99 latency in JVM-based services and summarise.",
    "permittedTools": [
      { "toolName": "web_search", "description": "Search for information on the web.", "argSchema": "..." },
      { "toolName": "summarize", "description": "Summarise a block of text.", "argSchema": "..." }
    ],
    "toolBudget": 4,
    "submittedBy": "operator-01",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "toolCalls": [
    {
      "callId": "tc-001",
      "toolName": "web_search",
      "argumentsJson": "{\"query\":\"JVM P99 latency causes\"}",
      "guardrailVerdict": "PERMITTED",
      "rejectionReason": null,
      "resultJson": "{\"results\":[\"GC pauses\",\"thread contention\",\"I/O wait\"]}",
      "calledAt": "2026-06-28T14:00:02Z",
      "completedAt": "2026-06-28T14:00:02Z"
    },
    {
      "callId": "tc-002",
      "toolName": "summarize",
      "argumentsJson": "{\"text\":\"GC pauses, thread contention, I/O wait ...\"}",
      "guardrailVerdict": "PERMITTED",
      "rejectionReason": null,
      "resultJson": "{\"summary\":\"Top 3 causes: GC pauses, thread contention, and I/O wait.\"}",
      "calledAt": "2026-06-28T14:00:03Z",
      "completedAt": "2026-06-28T14:00:03Z"
    }
  ],
  "result": {
    "answer": "Top 3 causes of P99 latency in JVM services: garbage-collection pauses, thread contention, and I/O wait. GC pauses dominate.",
    "toolCallsUsed": 2,
    "completedAt": "2026-06-28T14:00:04Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:04Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `toolCalls` is an empty array `[]` until the first tool call is requested.

A blocked tool call entry:

```json
{
  "callId": "tc-003",
  "toolName": "fetch_url",
  "argumentsJson": "{\"url\":\"https://example.com\"}",
  "guardrailVerdict": "BLOCKED",
  "rejectionReason": "tool-not-permitted: fetch_url is not in the permitted-tools list for this run",
  "resultJson": null,
  "calledAt": "2026-06-28T14:00:05Z",
  "completedAt": null
}
```

### SSE event format

```
event: run-update
data: { "runId": "run-3ae...", "status": "RUNNING", "toolCalls": [...], "result": null, ... }
```

One event per state transition (`ACCEPTED`, `RUNNING`, `COMPLETED`, `BLOCKED`, `FAILED`) and one event per tool-call state change (`ToolCallPermitted`, `ToolCallBlocked`, `ToolResultRecorded`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
