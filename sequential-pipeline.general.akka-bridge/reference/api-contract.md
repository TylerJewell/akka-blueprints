# API contract — akka-bridge

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `GraphEndpoint` → `GraphRunEntity` |
| `GET` | `/api/runs` | — | `200 [ GraphRunRecord... ]` (newest-first) | `GraphEndpoint` ← `GraphRunView` |
| `GET` | `/api/runs/{id}` | — | `200 GraphRunRecord` / `404` | `GraphEndpoint` ← `GraphRunView` |
| `GET` | `/api/runs/sse` | — | `text/event-stream` | `GraphEndpoint` ← `GraphRunView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitRunRequest (request body)

```json
{
  "graphId": "summarize-and-classify"
}
```

### GraphRunRecord (response body)

```json
{
  "runId": "r-7bc3e1...",
  "graphId": "summarize-and-classify",
  "plan": {
    "nodes": [
      {
        "nodeId": "summarize-001",
        "nodeType": "summarize",
        "description": "Summarize the input document into 3 key points.",
        "estimatedCostTokens": 800
      },
      {
        "nodeId": "classify-001",
        "nodeType": "classify",
        "description": "Classify the summarized key points by topic category.",
        "estimatedCostTokens": 400
      }
    ],
    "graphId": "summarize-and-classify",
    "plannedAt": "2026-06-28T10:00:00Z"
  },
  "nodeOutputs": {
    "outputs": [
      {
        "nodeId": "summarize-001",
        "rawOutput": "Key points: 1) Cost reduction 2) Adoption growth 3) Model diversity.",
        "outputSummary": "Three key points extracted.",
        "guardrailPassed": true,
        "executedAt": "2026-06-28T10:00:05Z"
      }
    ],
    "collectedAt": "2026-06-28T10:00:08Z"
  },
  "result": {
    "title": "Graph summarize-and-classify, run 2026-06-28",
    "summary": "The graph extracted three key points and classified them by topic.",
    "items": [
      {
        "nodeId": "summarize-001",
        "heading": "Document summarization",
        "body": "The summarization node extracted three key points from the source document.",
        "sourceNodeIds": ["summarize-001"]
      }
    ],
    "finalizedAt": "2026-06-28T10:00:12Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Node coverage, output integrity, reference provenance, and output parity all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:12Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:12Z",
  "violations": [
    {
      "nodeId": "summarize-001",
      "violationType": "prohibited-content",
      "reason": "content-violation: response matched prohibited pattern 'IGNORE PREVIOUS INSTRUCTIONS'",
      "blockedAt": "2026-06-28T10:00:03Z"
    }
  ]
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `violations` is an array — empty on the happy path, populated when the guardrail blocked a policy-violating node output.

### SSE event format

```
event: run-update
data: { "runId": "r-7bc3e1...", "status": "EXECUTED", "nodeOutputs": { ... }, ... }
```

One event per state transition (`CREATED`, `PLANNING`, `PLANNED`, `EXECUTING`, `EXECUTED`, `FINALIZING`, `FINALIZED`, `EVALUATED`, `FAILED`) and one per `GuardrailViolated` audit event:

```
event: run-violation
data: { "runId": "r-7bc3e1...", "nodeId": "summarize-001", "violationType": "prohibited-content", "reason": "...", "blockedAt": "..." }
```

Clients reconcile by `runId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitRunRequest` record and the `GraphRunCreated` event to carry it.
