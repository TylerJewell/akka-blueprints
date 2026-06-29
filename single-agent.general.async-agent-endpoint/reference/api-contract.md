# API contract — async-agent-endpoint

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `RunEndpoint` → `AgentRunEntity` |
| `GET` | `/api/runs` | — | `200 [ AgentRun... ]` (newest-first) | `RunEndpoint` ← `RunView` |
| `GET` | `/api/runs/{id}` | — | `200 AgentRun` / `404` | `RunEndpoint` ← `RunView` |
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
  "promptText": "Count the number of vowels in the string 'supercalifragilistic'",
  "submittedBy": "developer-42"
}
```

### AgentRun (response body)

```json
{
  "runId": "run-9c3f...",
  "prompt": {
    "taskId": "run-9c3f...",
    "promptText": "Count the number of vowels in the string 'supercalifragilistic'",
    "submittedBy": "developer-42",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "result": {
    "stdout": "",
    "stderr": "",
    "outputValue": "9",
    "guardrailHitOnAnyIteration": false,
    "code": {
      "code": "s = 'supercalifragilistic'\nvowels = sum(1 for c in s if c in 'aeiouAEIOU')\nvowels",
      "language": "python"
    },
    "completedAt": "2026-06-28T14:00:08Z"
  },
  "failure": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:08Z"
}
```

A `FAILED` run replaces `result` with a `failure` object:

```json
{
  "runId": "run-7b2a...",
  "prompt": { "promptText": "...", "submittedBy": "developer-42", "submittedAt": "..." },
  "result": null,
  "failure": {
    "kind": "GUARDRAIL_EXHAUSTED",
    "message": "All 3 iterations produced policy-violating code (subprocess import).",
    "failedAt": "2026-06-28T14:01:15Z"
  },
  "status": "FAILED",
  "createdAt": "2026-06-28T14:01:00Z",
  "finishedAt": "2026-06-28T14:01:15Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: run-update
data: { "runId": "run-9c3f...", "status": "COMPLETED", "result": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `COMPLETED`, `FAILED`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
