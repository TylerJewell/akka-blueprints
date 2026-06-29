# API contract — sdlc-task-planner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/plans` | `{ "prompt": String, "requestedBy"?: String }` | `202 { "planId": String }` | `PlanEndpoint` → `RequestQueue` |
| `GET` | `/api/plans` | — | `200 [ Plan... ]` | `PlanEndpoint` ← `PlanView` |
| `GET` | `/api/plans/{id}` | — | `200 Plan` / `404` | `PlanEndpoint` ← `PlanEntity` |
| `GET` | `/api/plans/sse` | — | `text/event-stream` (one event per plan change) | `PlanEndpoint` ← `PlanView` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `PlanEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `PlanEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `PlanEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Plan (full form, returned by `GET /api/plans/{id}`)

```json
{
  "planId": "p-3a1…",
  "prompt": "Analyse, design, implement, and review a feature that adds rate-limiting to a REST endpoint.",
  "status": "EXECUTING",
  "ledger": {
    "requirements": ["the system must limit each client to N requests per minute"],
    "constraints": ["all code changes must stay within /workspace/"],
    "plan": [
      "Analyst — extract acceptance criteria from requirements fixtures",
      "Architect — decide component structure for the rate limiter",
      "Coder — draft the RateLimiter class diff",
      "Reviewer — check the diff against security and coverage criteria"
    ],
    "currentDispatch": {
      "specialist": "CODER",
      "subtask": "Draft RateLimiter.java implementing token-bucket logic",
      "rationale": "Requirements and design are ready; implementation can start."
    }
  },
  "progress": {
    "entries": [
      {
        "attempt": 1,
        "specialist": "ANALYST",
        "subtask": "Extract acceptance criteria for rate-limiting feature",
        "verdict": "OK",
        "scrubbedResult": "Feature: rate-limiting. Criterion 1: reject after N req/min...",
        "blocker": null,
        "recordedAt": "2026-06-28T10:11:02Z"
      },
      {
        "attempt": 1,
        "specialist": "ARCHITECT",
        "subtask": "Decide component structure for the rate limiter",
        "verdict": "OK",
        "scrubbedResult": "Component: RateLimiter. Receives: HTTP request. Emits: allow/deny decision...",
        "blocker": null,
        "recordedAt": "2026-06-28T10:11:18Z"
      }
    ]
  },
  "deliverable": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T10:10:55Z",
  "finishedAt": null
}
```

### Plan (list form, returned by `GET /api/plans`)

The list form is `PlanRow` — same fields as `Plan`, but `progress.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full plan by id on row expand.

### SSE event format

```
event: plan-update
data: { "planId": "p-3a1…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `planId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "reviewing Coder output before continuing", "haltedAt": "2026-06-28T10:13:45Z" }
```
