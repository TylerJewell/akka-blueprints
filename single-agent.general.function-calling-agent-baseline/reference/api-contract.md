# API contract — function-calling-agent-baseline

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `AgentRunEndpoint` → `AgentRunEntity` + `AgentRunWorkflow` |
| `GET` | `/api/runs` | — | `200 [ AgentRun... ]` (newest-first) | `AgentRunEndpoint` ← `AgentRunView` |
| `GET` | `/api/runs/{id}` | — | `200 AgentRun` / `404` | `AgentRunEndpoint` ← `AgentRunView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `AgentRunEndpoint` ← `AgentRunView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "query": "If a train travels at 60 km/h for 2.5 hours and then at 80 km/h for 1.5 hours, what is the total distance?",
  "enabledTools": [
    {
      "toolName": "calculator",
      "description": "Evaluates a simple arithmetic expression and returns the numeric result.",
      "parameterTypes": { "expression": "string" }
    },
    {
      "toolName": "dictionary",
      "description": "Returns the definition of a single English word.",
      "parameterTypes": { "word": "string" }
    },
    {
      "toolName": "date-formatter",
      "description": "Formats an ISO-8601 date string into a human-readable form.",
      "parameterTypes": { "isoDate": "string", "format": "string" }
    }
  ],
  "submittedBy": "user-42"
}
```

### AgentRun (response body)

```json
{
  "runId": "run-7a9e...",
  "request": {
    "runId": "run-7a9e...",
    "query": "If a train travels at 60 km/h for 2.5 hours and then at 80 km/h for 1.5 hours, what is the total distance?",
    "enabledTools": [
      { "toolName": "calculator", "description": "...", "parameterTypes": { "expression": "string" } }
    ],
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T09:00:00Z"
  },
  "answer": {
    "answer": "The total distance is 270 km. The first leg covers 150 km (60 × 2.5) and the second leg covers 120 km (80 × 1.5).",
    "toolCallTrace": [
      {
        "iteration": 1,
        "toolName": "calculator",
        "arguments": { "expression": "60*2.5" },
        "result": "150.0",
        "blocked": false,
        "calledAt": "2026-06-28T09:00:03Z"
      },
      {
        "iteration": 2,
        "toolName": "calculator",
        "arguments": { "expression": "80*1.5" },
        "result": "120.0",
        "blocked": false,
        "calledAt": "2026-06-28T09:00:05Z"
      },
      {
        "iteration": 3,
        "toolName": "calculator",
        "arguments": { "expression": "150+120" },
        "result": "270.0",
        "blocked": false,
        "calledAt": "2026-06-28T09:00:07Z"
      }
    ],
    "totalIterations": 3,
    "guardrailSummary": {
      "toolCallBlocked": false,
      "answerBlocked": false,
      "toolCallBlockCount": 0,
      "answerBlockCount": 0
    },
    "answeredAt": "2026-06-28T09:00:08Z"
  },
  "status": "ANSWER_RECORDED",
  "createdAt": "2026-06-28T09:00:00Z",
  "finishedAt": "2026-06-28T09:00:08Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### SSE event format

```
event: run-update
data: { "runId": "run-7a9e...", "status": "ANSWER_RECORDED", "answer": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `ANSWER_RECORDED`, `FAILED`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
