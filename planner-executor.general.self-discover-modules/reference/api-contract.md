# API contract — self-discover-modules

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "taskId": String }` | `TaskEndpoint` → `RequestQueue` |
| `GET` | `/api/tasks` | — | `200 [ TaskRow... ]` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `TaskEndpoint` ← `TaskEntity` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `TaskEndpoint` ← `TaskView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `TaskEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `TaskEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `TaskEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Task (full form, returned by `GET /api/tasks/{id}`)

```json
{
  "taskId": "t-9a1…",
  "prompt": "Explain why gradient descent can converge to a local minimum rather than a global minimum.",
  "status": "SOLVING",
  "structure": {
    "selectedModules": ["decomposition", "causal-reasoning", "analogy-mapping", "verification", "synthesis"],
    "adaptedSteps": [
      {
        "moduleName": "decomposition",
        "adaptedDescription": "Break the claim into its components: what gradient descent does at each step, what local vs. global minima are, and under what conditions convergence to local minima occurs.",
        "role": "DECOMPOSE",
        "stepIndex": 0
      },
      {
        "moduleName": "causal-reasoning",
        "adaptedDescription": "Trace the causal chain from a random start point through gradient steps to explain why the algorithm follows the local gradient rather than the global landscape.",
        "role": "ANALYZE",
        "stepIndex": 1
      },
      {
        "moduleName": "synthesis",
        "adaptedDescription": "Integrate the causal explanation and the analogy into a coherent answer that addresses the user's prompt directly.",
        "role": "SYNTHESIZE",
        "stepIndex": 4
      }
    ],
    "compositionRationale": "Causal reasoning before verification ensures the core intuition is established before edge cases are tested.",
    "revisionNumber": null
  },
  "lastEval": {
    "passed": true,
    "score": 0.85,
    "rejectionReason": null
  },
  "stepExecutions": [
    {
      "stepIndex": 0,
      "moduleName": "decomposition",
      "observation": "Gradient descent updates parameters in the direction of steepest descent. A local minimum has zero gradient but is not necessarily the global lowest point. Non-convex loss surfaces have many such local minima.",
      "ok": true,
      "errorReason": null,
      "executedAt": "2026-06-28T10:12:03Z"
    }
  ],
  "answer": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:11:55Z",
  "finishedAt": null
}
```

### Task (list form, returned by `GET /api/tasks`)

The list form is `TaskRow` — same fields as `Task`, but `stepExecutions` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `observation` is capped at 240 characters. The UI fetches the full task by id on click via `GET /api/tasks/{id}`.

### SSE event format

```
event: task-update
data: { "taskId": "t-9a1…", "status": "SOLVING", ... }
```

One event per state transition. Clients reconcile by `taskId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "pausing for review", "haltedAt": "2026-06-28T10:13:44Z" }
```
