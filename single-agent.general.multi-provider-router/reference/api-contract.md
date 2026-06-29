# API contract — multi-provider-router

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/calls` | `SubmitCallRequest` | `201 { callId }` | `RouterEndpoint` → `CallEntity` + `RoutingWorkflow` |
| `GET` | `/api/calls` | — | `200 [ CallRow... ]` (newest-first) | `RouterEndpoint` ← `CallView` |
| `GET` | `/api/calls/{id}` | — | `200 CallRow` / `404` | `RouterEndpoint` ← `CallView` |
| `GET` | `/api/calls/sse` | — | `text/event-stream` | `RouterEndpoint` ← `CallView` |
| `GET` | `/api/monitor` | — | `200 [ MonitorRow... ]` (newest-first) | `RouterEndpoint` ← `MonitorView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `RouterEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `RouterEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `RouterEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCallRequest (request body)

```json
{
  "promptText": "What is the capital of Portugal?",
  "hint": "AUTO",
  "submittedBy": "developer-01"
}
```

`hint` must be one of `"AUTO"`, `"OPENAI"`, or `"ANTHROPIC"`. `AUTO` lets the router's shuffle strategy decide.

### CallRow (response body)

```json
{
  "callId": "c-3ae...",
  "request": {
    "callId": "c-3ae...",
    "promptText": "What is the capital of Portugal?",
    "hint": "AUTO",
    "submittedBy": "developer-01",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "result": {
    "selectedProvider": "ANTHROPIC",
    "responseText": "The capital of Portugal is Lisbon.",
    "latencyMs": 843,
    "completedAt": "2026-06-28T10:00:00.843Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:00.843Z",
  "errorReason": null
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### MonitorRow (response body)

```json
{
  "windowId": "window-3",
  "windowSize": 5,
  "stats": [
    {
      "provider": "OPENAI",
      "totalCalls": 3,
      "failedCalls": 0,
      "p50LatencyMs": 612,
      "p95LatencyMs": 1240,
      "avgQualityScore": 4.3
    },
    {
      "provider": "ANTHROPIC",
      "totalCalls": 2,
      "failedCalls": 0,
      "p50LatencyMs": 780,
      "p95LatencyMs": 990,
      "avgQualityScore": 4.8
    }
  ],
  "generatedAt": "2026-06-28T10:01:00Z"
}
```

### SSE event format

```
event: call-update
data: { "callId": "c-3ae...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`PENDING`, `DISPATCHED`, `COMPLETED`, `FAILED`). Clients reconcile by `callId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
