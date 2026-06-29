# API contract — react-loop-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `RunEndpoint` → `RunEntity` |
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
  "query": "What is (14 + 8) * 3, and what is the cube root of that result rounded to 2 decimal places?",
  "availableTools": [
    {
      "toolName": "Calculator",
      "description": "Evaluates a safe arithmetic expression and returns the numeric result.",
      "paramSchema": "{ \"expression\": \"<arithmetic string>\" }"
    },
    {
      "toolName": "WebLookup",
      "description": "Returns a canned paragraph on a topic keyword.",
      "paramSchema": "{ \"keyword\": \"<search term>\" }"
    }
  ],
  "deniedTools": ["DataFetch"],
  "submittedBy": "user-42"
}
```

### Run (response body)

```json
{
  "runId": "r-9fa...",
  "request": {
    "runId": "r-9fa...",
    "query": "What is (14 + 8) * 3 ...",
    "availableTools": [
      { "toolName": "Calculator", "description": "...", "paramSchema": "..." }
    ],
    "deniedTools": ["DataFetch"],
    "submittedBy": "user-42",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "steps": [
    {
      "stepIndex": 0,
      "kind": "THOUGHT",
      "content": "I need to compute 14+8 first.",
      "toolName": null,
      "blocked": false,
      "recordedAt": "2026-06-28T14:00:01Z"
    },
    {
      "stepIndex": 1,
      "kind": "ACTION",
      "content": "{ \"toolName\": \"Calculator\", \"params\": { \"expression\": \"14 + 8\" } }",
      "toolName": "Calculator",
      "blocked": false,
      "recordedAt": "2026-06-28T14:00:02Z"
    },
    {
      "stepIndex": 2,
      "kind": "OBSERVATION",
      "content": "22",
      "toolName": "Calculator",
      "blocked": false,
      "recordedAt": "2026-06-28T14:00:02Z"
    },
    {
      "stepIndex": 5,
      "kind": "ANSWER",
      "content": "66. Cube root is approximately 4.04. (Calculator used.)",
      "toolName": null,
      "blocked": false,
      "recordedAt": "2026-06-28T14:00:05Z"
    }
  ],
  "result": {
    "finalAnswer": "66. Cube root is approximately 4.04. (Calculator used.)",
    "steps": [ "... same as above ..." ],
    "outcome": "ANSWERED",
    "decidedAt": "2026-06-28T14:00:05Z"
  },
  "eval": {
    "score": 5,
    "rationale": "All tool claims are supported by observations; no circular thoughts detected.",
    "evaluatedAt": "2026-06-28T14:00:05Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:05Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`). `steps` is always an array — empty `[]` before any steps are recorded, never `null`.

A blocked action appears in the steps array with `blocked: true`:

```json
{
  "stepIndex": 3,
  "kind": "ACTION",
  "content": "{ \"toolName\": \"DataFetch\", \"params\": { \"recordId\": \"R-42\" } }",
  "toolName": "DataFetch",
  "blocked": true,
  "recordedAt": "2026-06-28T14:00:03Z"
}
```

### SSE event format

```
event: run-update
data: { "runId": "r-9fa...", "status": "RUNNING", "steps": [...], ... }
```

One event per entity transition: `PENDING`, `RUNNING`, each `StepRecorded` / `ActionBlocked`, `COMPLETED`, `EXHAUSTED`, `FAILED`, and the `EvaluationScored` append. Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay prior steps.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity should wrap the endpoint with their auth middleware and populate `submittedBy` from the authenticated principal rather than the request body.
