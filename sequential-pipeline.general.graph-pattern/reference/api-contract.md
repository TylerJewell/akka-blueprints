# API contract — graph-pattern

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/runs` | `SubmitRunRequest` | `201 { runId }` | `GraphEndpoint` → `GraphRunEntity` |
| `GET` | `/api/runs` | — | `200 [ RunRecord... ]` (newest-first) | `GraphEndpoint` ← `GraphRunView` |
| `GET` | `/api/runs/{id}` | — | `200 RunRecord` / `404` | `GraphEndpoint` ← `GraphRunView` |
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
  "description": "Summarize recent Akka release notes"
}
```

### RunRecord (response body)

```json
{
  "runId": "r-3b7e91...",
  "description": "Summarize recent Akka release notes",
  "parsedRequest": {
    "intent": "Summarize the most recent Akka release notes into a readable digest",
    "constraints": ["focus on breaking changes", "include version numbers"],
    "parsedAt": "2026-06-28T10:00:00Z"
  },
  "plan": {
    "nodes": [
      { "nodeId": "fetch-releases", "label": "Fetch release notes", "predecessorIds": [] },
      { "nodeId": "summarize", "label": "Summarize fetched notes", "predecessorIds": ["fetch-releases"] }
    ],
    "edges": [
      { "fromNodeId": "fetch-releases", "toNodeId": "summarize" }
    ],
    "plannedAt": "2026-06-28T10:00:03Z"
  },
  "executionResult": {
    "outputs": [
      {
        "nodeId": "fetch-releases",
        "outputId": "out-fetch-001",
        "body": "Akka 3.6.0 release notes retrieved: actor model updates, improved workflow timeout semantics, and new view streaming APIs.",
        "sourceNodeIds": []
      },
      {
        "nodeId": "summarize",
        "outputId": "out-sum-001",
        "body": "Akka 3.6.0 introduces improved workflow timeout handling and streaming view APIs. Actor model updates include enhanced supervision semantics.",
        "sourceNodeIds": ["fetch-releases"]
      }
    ],
    "executionOrder": ["fetch-releases", "summarize"],
    "executedAt": "2026-06-28T10:00:10Z"
  },
  "taskResult": {
    "title": "Akka release notes summary",
    "summary": "Two-node DAG processed release note fetching and summarisation. Output covers Akka 3.6.0 highlights.",
    "outputRefs": [
      { "nodeId": "fetch-releases", "outputId": "out-fetch-001" },
      { "nodeId": "summarize", "outputId": "out-sum-001" }
    ],
    "mergedAt": "2026-06-28T10:00:12Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Node coverage, output traceability, no phantom nodes, and ordering proof all satisfied.",
    "evaluatedAt": "2026-06-28T10:00:13Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T09:59:58Z",
  "finishedAt": "2026-06-28T10:00:13Z",
  "dependencyViolations": []
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`). `dependencyViolations` is an array — empty on the happy path, populated when the agent attempted an out-of-order node execution or a misordered phase tool call.

### SSE event format

```
event: run-update
data: { "runId": "r-3b7e91...", "status": "EXECUTING", "plan": { ... }, ... }
```

One event per status transition (`CREATED`, `PARSING`, `PARSED`, `PLANNING`, `PLANNED`, `EXECUTING`, `EXECUTED`, `MERGING`, `MERGED`, `EVALUATED`, `FAILED`) and one per `NodeExecuted` audit event:

```
event: node-executed
data: { "runId": "r-3b7e91...", "nodeId": "fetch-releases", "outputId": "out-fetch-001", "executedAt": "..." }
```

And one per `DependencyViolated` audit event:

```
event: dependency-violation
data: { "runId": "r-3b7e91...", "phase": "EXECUTE", "nodeId": "summarize", "missingPredecessors": ["fetch-releases"], "reason": "dependency-violation: node summarize requires predecessors [fetch-releases] but fetch-releases not yet executed", "violatedAt": "..." }
```

Clients reconcile by `runId`; a `run-update` event always carries the full row at the moment of transition, so a late-joining client never needs to replay. `node-executed` and `dependency-violation` events carry only their specific payload and must be merged into the client's cached run row.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and stamp a `submittedBy` field from the authenticated principal — extend the `SubmitRunRequest` record and the `RunCreated` event to carry it.
