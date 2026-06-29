# API contract — planner-executor-general-planner-executor

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/plans` | `{ "goal": String, "requestedBy"?: String }` | `202 { "planId": String }` | `PlanEndpoint` → `GoalQueue` |
| `GET` | `/api/plans` | — | `200 [ PlanRow... ]` | `PlanEndpoint` ← `PlanView` |
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
  "planId": "p-3a9…",
  "goal": "Draft and send a weekly status report to the team.",
  "status": "EXECUTING",
  "ledger": {
    "goal": "Draft and send a weekly status report to the team.",
    "openSteps": [
      "Draft the report body",
      "Review draft against goal criteria",
      "Finalise and output the report text"
    ],
    "completedSteps": [
      "Gather completed items from the fixture data this week",
      "Identify blockers and upcoming milestones"
    ],
    "blockedSteps": [],
    "activeStep": "Draft the report body"
  },
  "stepLog": {
    "entries": [
      {
        "attempt": 1,
        "step": "Gather completed items from the fixture data this week",
        "verdict": "OK",
        "result": "Found 4 completed items in fixtures: feature-a shipped, bug-b fixed...",
        "blocker": null,
        "recordedAt": "2026-06-28T08:11:03Z"
      },
      {
        "attempt": 1,
        "step": "Identify blockers and upcoming milestones",
        "verdict": "OK",
        "result": "Two blockers noted: dependency on team-c deliverable, infra upgrade pending...",
        "blocker": null,
        "recordedAt": "2026-06-28T08:11:11Z"
      }
    ]
  },
  "outcome": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T08:10:57Z",
  "finishedAt": null
}
```

### PlanRow (list form, returned by `GET /api/plans`)

`PlanRow` mirrors `Plan` but `stepLog.entries` is truncated to the last 3 entries plus `truncatedFromTotal: int`. Each entry's `result` is capped at 240 characters. The UI fetches the full plan by id on row expand.

### SSE event format

```
event: plan-update
data: { "planId": "p-3a9…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `planId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating unexpected step results", "haltedAt": "2026-06-28T08:13:45Z" }
```
