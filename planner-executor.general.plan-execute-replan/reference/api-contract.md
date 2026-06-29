# API contract — plan-execute-replan

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/goals` | `{ "goal": String, "requestedBy"?: String }` | `202 { "goalId": String }` | `GoalEndpoint` → `GoalQueue` |
| `GET` | `/api/goals` | — | `200 [ GoalRow... ]` | `GoalEndpoint` ← `GoalView` |
| `GET` | `/api/goals/{id}` | — | `200 Goal` / `404` | `GoalEndpoint` ← `GoalEntity` |
| `GET` | `/api/goals/sse` | — | `text/event-stream` (one event per goal change) | `GoalEndpoint` ← `GoalView` |
| `POST` | `/api/control/pause` | `{ "reason": String }` | `200 { "paused": true, "reason": String }` | `GoalEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "paused": false }` | `GoalEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "paused": boolean, "reason"?: String, "pausedAt"?: Instant }` | `GoalEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Goal (full form, returned by `GET /api/goals/{id}`)

```json
{
  "goalId": "g-4a1…",
  "goal": "Find the current Akka SDK version and list its top three new features.",
  "status": "EXECUTING",
  "plan": {
    "goalSummary": "Retrieve the current Akka SDK version and enumerate its top three new features.",
    "steps": [
      {
        "stepIndex": 0,
        "description": "Search for the latest Akka SDK release version.",
        "tool": { "kind": "SEARCH", "argument": "akka.io latest SDK release" },
        "expectedOutput": "version string and release date"
      },
      {
        "stepIndex": 1,
        "description": "Read the release notes fixture to find new features.",
        "tool": { "kind": "READ", "argument": "sample-data/release-notes.md" },
        "expectedOutput": "feature list for that version"
      },
      {
        "stepIndex": 2,
        "description": "Summarise findings into a three-bullet answer.",
        "tool": { "kind": "SUMMARISE", "argument": "summarise observations so far" },
        "expectedOutput": "three-bullet summary"
      }
    ]
  },
  "currentStepIndex": 1,
  "revisionCount": 0,
  "observations": {
    "entries": [
      {
        "stepIndex": 0,
        "type": "STEP_OK",
        "content": "Akka 3.6.0 released 2026-06-01 (akka.io, 'Release notes')...",
        "evalRecord": null,
        "recordedAt": "2026-06-28T10:12:03Z"
      },
      {
        "stepIndex": 0,
        "type": "EVAL",
        "content": "ReplanDecision: Continue(1)",
        "evalRecord": {
          "dimension": "goal-alignment",
          "score": 95,
          "rationale": "Plan steps align with goal; no repeated failures; conclusion not premature."
        },
        "recordedAt": "2026-06-28T10:12:05Z"
      }
    ]
  },
  "conclusion": null,
  "failureReason": null,
  "pauseReason": null,
  "createdAt": "2026-06-28T10:12:00Z",
  "finishedAt": null
}
```

### Goal (list form, returned by `GET /api/goals`)

The list form is `GoalRow` — same fields as `Goal`, but `observations.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>`. The UI fetches the full goal by id on row expand.

### SSE event format

```
event: goal-update
data: { "goalId": "g-4a1…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `goalId`. The SSE channel also emits `event: control-update` whenever the operator pause flag changes:

```
event: control-update
data: { "paused": true, "reason": "reviewing low eval scores", "pausedAt": "2026-06-28T10:14:20Z" }
```
