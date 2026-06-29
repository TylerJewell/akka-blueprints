# API contract — akka-resumable-agent-http

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/agent/run` | `StartRunRequest` | `201 { runId }` | `AgentEndpoint` → `AgentRunEntity` |
| `GET` | `/api/agent/runs` | — | `200 [ AgentRun... ]` (newest-first) | `AgentEndpoint` ← `AgentRunView` |
| `GET` | `/api/agent/instances/{id}` | — | `200 AgentRun` / `404` | `AgentEndpoint` ← `AgentRunView` |
| `GET` | `/api/agent/runs/sse` | — | `text/event-stream` | `AgentEndpoint` ← `AgentRunView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartRunRequest (request body)

```json
{
  "location": "San Francisco, CA",
  "queryType": "CURRENT",
  "requestedBy": "dev-user-01"
}
```

`queryType` must be one of `CURRENT`, `FORECAST`, `AIR_QUALITY`.

### AgentRun (response body — completed, resumed example)

```json
{
  "runId": "run-a3f...",
  "request": {
    "runId": "run-a3f...",
    "location": "San Francisco, CA",
    "queryType": "CURRENT",
    "requestedBy": "dev-user-01",
    "requestedAt": "2026-06-28T15:00:00Z"
  },
  "toolCall": {
    "toolName": "SlowWeatherTool",
    "argumentLocation": "San Francisco, CA",
    "queryType": "CURRENT",
    "calledAt": "2026-06-28T15:00:01Z",
    "guardrailRejected": false,
    "rejectionReason": null
  },
  "report": {
    "location": "San Francisco, CA",
    "queryType": "CURRENT",
    "summary": "Foggy morning clearing to sunshine by afternoon.",
    "temperatureCelsius": 17.4,
    "conditions": "Fog then Sunny",
    "reportedAt": "2026-06-28T15:00:10Z"
  },
  "incident": {
    "runId": "run-a3f...",
    "interruptedStep": "toolCallStep",
    "elapsedBeforeCrashMs": 4210,
    "severity": "WARNING",
    "notes": "Process killed while SlowWeatherTool was executing. Workflow resumed from toolCallStep checkpoint.",
    "detectedAt": "2026-06-28T15:01:45Z"
  },
  "status": "COMPLETED",
  "resumeCount": 1,
  "createdAt": "2026-06-28T15:00:00Z",
  "finishedAt": "2026-06-28T15:01:55Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### AgentRun (response body — guardrail rejection example)

```json
{
  "runId": "run-b7c...",
  "toolCall": {
    "toolName": "SlowWeatherTool",
    "argumentLocation": "",
    "queryType": "CURRENT",
    "calledAt": "2026-06-28T15:05:00Z",
    "guardrailRejected": true,
    "rejectionReason": "location-empty"
  },
  "status": "TOOL_CALLED",
  "resumeCount": 0,
  "createdAt": "2026-06-28T15:05:00Z",
  "finishedAt": null
}
```

### SSE event format

```
event: run-update
data: { "runId": "run-a3f...", "status": "RESUMED", "resumeCount": 1, "incident": { ... }, ... }
```

One event per state transition (`INITIATED`, `TOOL_CALLED`, `REPORTING`, `RESUMED`, `COMPLETED`, `FAILED`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `requestedBy` from the authenticated principal rather than the request body.
