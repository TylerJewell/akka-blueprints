# API contract — self-discover-planner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/solves` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "solveId": String }` | `SolveEndpoint` → `RequestQueue` |
| `GET` | `/api/solves` | — | `200 [ SolveRow... ]` | `SolveEndpoint` ← `SolveView` |
| `GET` | `/api/solves/{id}` | — | `200 Solve` / `404` | `SolveEndpoint` ← `SolveEntity` |
| `GET` | `/api/solves/sse` | — | `text/event-stream` (one event per solve change) | `SolveEndpoint` ← `SolveView` |
| `GET` | `/api/registry` | — | `200 { "modules": [ ReasoningModule... ] }` | `SolveEndpoint` ← `ModuleRegistry` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Solve (full form, returned by `GET /api/solves/{id}`)

```json
{
  "solveId": "s-9a1…",
  "prompt": "What are the tradeoffs between consistency and availability in distributed databases?",
  "status": "EXECUTING",
  "plan": {
    "selectedModuleIds": ["m-decompose-01", "m-analyse-02", "m-compare-01", "m-verify-01", "m-reflect-01"],
    "stepOrder": [
      { "index": 0, "moduleId": "m-decompose-01", "objective": "Break the topic into evaluation sub-questions", "inputsFrom": [] },
      { "index": 1, "moduleId": "m-analyse-02", "objective": "Analyse each consistency model against sub-questions", "inputsFrom": [0] },
      { "index": 2, "moduleId": "m-compare-01", "objective": "Compare strong, eventual, and causal consistency", "inputsFrom": [1] },
      { "index": 3, "moduleId": "m-verify-01", "objective": "Verify there are no contradictions in the comparison", "inputsFrom": [2] },
      { "index": 4, "moduleId": "m-reflect-01", "objective": "Reflect on alternative framings", "inputsFrom": [3] }
    ],
    "rationale": "A DECOMPOSE step is needed to ground the analysis in concrete criteria. VERIFY and REFLECT ensure the comparison is internally consistent and alternative perspectives are surfaced.",
    "revisionNote": null
  },
  "planEval": {
    "score": 0.88,
    "verdict": "PASS",
    "feedback": "Plan covers all required dimensions and data flow is coherent."
  },
  "executionLog": {
    "steps": [
      {
        "stepIndex": 0,
        "moduleId": "m-decompose-01",
        "kind": "DECOMPOSE",
        "objective": "Break the topic into evaluation sub-questions",
        "result": {
          "moduleId": "m-decompose-01",
          "kind": "DECOMPOSE",
          "output": "1. What does strong consistency guarantee?\n2. What does eventual consistency sacrifice?\n3. Under what network conditions does each model fail?\n4. What are the latency implications of each?",
          "ok": true,
          "errorReason": null
        },
        "executedAt": "2026-06-28T09:11:04Z"
      }
    ]
  },
  "answer": null,
  "failureReason": null,
  "planRevisionCount": 0,
  "createdAt": "2026-06-28T09:10:58Z",
  "finishedAt": null
}
```

### SolveRow (list form, returned by `GET /api/solves`)

`SolveRow` mirrors `Solve` but `executionLog.steps` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. Each `result.output` is capped at 240 characters. The UI fetches the full solve by id on row expand.

### ReasoningModule

```json
{
  "moduleId": "m-decompose-01",
  "kind": "DECOMPOSE",
  "name": "Problem Decomposer",
  "description": "Breaks a complex question into 3–6 sub-questions or sub-problems that can be addressed independently."
}
```

### SSE event format

```
event: solve-update
data: { "solveId": "s-9a1…", "status": "EVALUATING", "planRevisionCount": 0, ... }
```

One event per state transition. Clients reconcile by `solveId`. The SSE channel emits events for every `SolveEntity` event type, including intermediate ones like `PlanRejected` and `ModuleExecuted`, so the UI can show real-time step progress.
