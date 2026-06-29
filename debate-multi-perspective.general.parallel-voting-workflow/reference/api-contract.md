# API contract — parallel-voting-workflow

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "title": String, "description": String, "submittedBy"?: String }` | `202 { "taskId": String }` | `TaskEndpoint` → `TaskQueue` |
| `GET` | `/api/tasks` | — | `200 [ Task... ]` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks?status=DECIDED` | — | `200 [ Task... ]` (filtered client-side) | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 Task` / `404` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` (one event per task change) | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/classification-questions` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/survey-template` | — | `application/json` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Task

```json
{
  "taskId": "tk-4a1…",
  "title": "Migrate invoicing database to new schema",
  "status": "DECIDED",
  "description": "Move all invoice records from v2 to v3 schema over the weekend maintenance window …",
  "brief": {
    "feasibilityFocus": "Check whether the migration window and rollback plan are realistic.",
    "riskFocus": "Identify failure modes that could cause data loss or prolonged downtime.",
    "alignmentFocus": "Confirm the migration aligns with the zero-downtime service commitment."
  },
  "feasibility": {
    "dimension": "FEASIBILITY",
    "ballot": "APPROVE",
    "confidence": 4,
    "reasons": [ { "weight": "MEDIUM", "text": "Window is adequate for the stated record count." } ],
    "votedAt": "2026-06-28T09:11:02Z"
  },
  "risk": {
    "dimension": "RISK",
    "ballot": "APPROVE",
    "confidence": 3,
    "reasons": [ { "weight": "HIGH", "text": "Rollback plan is present but untested against production volume." } ],
    "votedAt": "2026-06-28T09:11:03Z"
  },
  "alignment": {
    "dimension": "ALIGNMENT",
    "ballot": "APPROVE",
    "confidence": 4,
    "reasons": [ { "weight": "LOW", "text": "Maintenance window is consistent with zero-downtime policy." } ],
    "votedAt": "2026-06-28T09:11:04Z"
  },
  "decision": {
    "decision": "PROCEED",
    "summary": "All three dimensions voted APPROVE. The risk voter flagged the untested rollback plan, which is noted in the decision but does not reach the threshold for a HOLD. Feasibility and alignment are both clean. The task may proceed with the recommendation that a rollback dry-run is conducted before the maintenance window.",
    "votes": [ { "dimension": "FEASIBILITY" }, { "dimension": "RISK" }, { "dimension": "ALIGNMENT" } ],
    "quorumMet": true,
    "decidedAt": "2026-06-28T09:11:09Z"
  },
  "failureReason": null,
  "alignmentScore": 4,
  "alignmentRationale": "PROCEED is justified by three APPROVE ballots; the summary correctly surfaces the medium-confidence risk concern.",
  "createdAt": "2026-06-28T09:10:55Z",
  "finishedAt": "2026-06-28T09:11:09Z"
}
```

Lifecycle fields (`description`, `brief`, `feasibility`, `risk`, `alignment`, `decision`, `failureReason`, `alignmentScore`, `alignmentRationale`, `finishedAt`) are `Optional<T>` in Java and serialize as the raw value or `null` — never an `{ "value": … }` wrapper (Lesson 6).

### SSE event format

```
event: task-update
data: { "taskId": "tk-4a1…", "status": "VOTING", ... }
```

One event per state transition. Clients reconcile by `taskId`. The `AlignmentScored` transition emits a `task-update` event whose status is unchanged but whose `alignmentScore` is now populated.
