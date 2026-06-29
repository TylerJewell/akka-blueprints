# API contract — inline-agent-demo

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `RunEndpoint` → `RunEntity`, `RunWorkflow` |
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
  "question": "What are the key principles of the CAP theorem?",
  "agentDefinition": {
    "agentName": "qa-assistant",
    "systemPrompt": "You are a technical Q&A assistant. Answer questions accurately and concisely. Cite sources where applicable.",
    "outputSchema": "{ \"type\": \"object\", \"properties\": { \"answer\": { \"type\": \"string\" }, \"tokenCount\": { \"type\": \"integer\" }, \"answeredAt\": { \"type\": \"string\", \"format\": \"date-time\" } }, \"required\": [\"answer\", \"tokenCount\", \"answeredAt\"] }",
    "allowedTools": []
  },
  "submittedBy": "demo-user-1"
}
```

### Run (response body)

```json
{
  "runId": "r-4ac...",
  "request": {
    "runId": "r-4ac...",
    "question": "What are the key principles of the CAP theorem?",
    "agentDefinition": {
      "agentName": "qa-assistant",
      "systemPrompt": "You are a technical Q&A assistant...",
      "outputSchema": "{ ... }",
      "allowedTools": []
    },
    "submittedBy": "demo-user-1",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "response": {
    "answer": "The CAP theorem states that a distributed system can guarantee at most two of three properties: Consistency, Availability, and Partition tolerance. In practice, because network partitions cannot be avoided, designers must choose between consistency and availability under partition.",
    "tokenCount": 58,
    "answeredAt": "2026-06-28T10:00:12Z"
  },
  "failureReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:12Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### Failed run (response body)

```json
{
  "runId": "r-7bd...",
  "request": { ... },
  "response": null,
  "failureReason": "Agent definition is missing required field: systemPrompt",
  "status": "FAILED",
  "createdAt": "2026-06-28T10:01:00Z",
  "finishedAt": "2026-06-28T10:01:00Z"
}
```

### SSE event format

```
event: run-update
data: { "runId": "r-4ac...", "status": "COMPLETED", "response": { ... }, ... }
```

One event per state transition (`RECEIVED`, `RUNNING`, `COMPLETED`, `FAILED`). Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
