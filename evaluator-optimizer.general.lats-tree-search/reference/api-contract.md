# API contract — lats-tree-search

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/trees` | `{ "taskDescription": String, "nodeBudget"?: Integer, "submittedBy"?: String }` | `202 { "treeId": String }` | `SearchEndpoint` → `ProblemQueue` |
| `GET` | `/api/trees` | — | `200 [ SearchTree... ]` (optional `?status=EXPANDING\|REFLECTING\|SOLVED\|BUDGET_EXHAUSTED` — filtered client-side from `getAllTrees`) | `SearchEndpoint` ← `TreeView` |
| `GET` | `/api/trees/{id}` | — | `200 SearchTree` / `404` | `SearchEndpoint` ← `TreeView` |
| `GET` | `/api/trees/sse` | — | `text/event-stream` (one event per tree change) | `SearchEndpoint` ← `TreeView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/trees`

- Missing `nodeBudget` → 20 (from `lats-tree-search.search.node-budget`).
- Missing `submittedBy` → `"anonymous"`.
- `nodeBudget` must be in `[3, 200]`; otherwise `400`.
- Duplicate-detection window: 10 s on `(taskDescription, submittedBy)`; the second submission returns the first `treeId` (200) instead of starting a new workflow.

## JSON shapes

### SearchTree

```json
{
  "treeId": "t-8a3f1…",
  "taskDescription": "Design a caching strategy for a read-heavy API.",
  "nodeBudget": 20,
  "expansionWidth": 3,
  "status": "SOLVED",
  "nodes": [
    {
      "nodeId": "root",
      "parentNodeId": null,
      "depth": 0,
      "actionDescription": "",
      "score": null,
      "nodeStatus": "SELECTED"
    },
    {
      "nodeId": "root-c",
      "parentNodeId": "root",
      "depth": 1,
      "actionDescription": "Partition the cache by data ownership domain.",
      "score": {
        "candidateId": "root-c",
        "score": 7,
        "isTerminal": false,
        "note": {
          "rationale": "Relevant and specific; completeness is low — no cache store named.",
          "actionability": "Expand by selecting a specific cache store and TTL policy."
        },
        "backpropDelta": 0.7,
        "reflectedAt": "2026-06-28T09:01:20Z"
      },
      "nodeStatus": "SELECTED"
    },
    {
      "nodeId": "root-c-b",
      "parentNodeId": "root-c",
      "depth": 2,
      "actionDescription": "Use Redis Cluster with per-domain keyspace prefixes and TTL-based invalidation.",
      "score": {
        "candidateId": "root-c-b",
        "score": 9,
        "isTerminal": true,
        "note": {
          "rationale": "All rubric dimensions at 9; self-sufficient terminal solution.",
          "actionability": "No further expansion needed."
        },
        "backpropDelta": 0.9,
        "reflectedAt": "2026-06-28T09:02:05Z"
      },
      "nodeStatus": "TERMINAL"
    }
  ],
  "bestPath": ["root", "root-c", "root-c-b"],
  "terminalNodeId": "root-c-b",
  "terminalContent": "Use Redis Cluster with per-domain keyspace prefixes and TTL-based invalidation.",
  "exhaustionReason": null,
  "createdAt": "2026-06-28T09:00:55Z",
  "finishedAt": "2026-06-28T09:02:06Z"
}
```

### SSE event format

```
event: tree-update
data: { "treeId": "t-8a3f1…", "status": "REFLECTING", "nodes": [...], "bestPath": [...], ... }
```

One event per state transition. Clients reconcile by `treeId`. The full `SearchTree` JSON is included so a fresh client can render the row without a separate fetch.
