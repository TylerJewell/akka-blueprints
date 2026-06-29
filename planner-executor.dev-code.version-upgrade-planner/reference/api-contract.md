# API contract — version-upgrade-planner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/jobs` | `{ "sourceVersion": String, "targetVersion": String, "requestedBy"?: String }` | `202 { "jobId": String }` | `UpgradeJobEndpoint` → `UpgradeRequestQueue` |
| `GET` | `/api/jobs` | — | `200 [ UpgradeJobRow... ]` | `UpgradeJobEndpoint` ← `UpgradeJobView` |
| `GET` | `/api/jobs/{id}` | — | `200 UpgradeJob` / `404` | `UpgradeJobEndpoint` ← `UpgradeJobEntity` |
| `GET` | `/api/jobs/sse` | — | `text/event-stream` (one event per job change) | `UpgradeJobEndpoint` ← `UpgradeJobView` |
| `GET` | `/api/approvals/{approvalId}` | — | `200 ApprovalRequest` / `404` | `UpgradeJobEndpoint` ← `ApprovalEntity` |
| `POST` | `/api/approvals/{approvalId}/approve` | `{ "reviewedBy": String, "comment"?: String }` | `200 { "approvalId": String, "approved": true }` | `UpgradeJobEndpoint` → `ApprovalEntity` |
| `POST` | `/api/approvals/{approvalId}/reject` | `{ "reviewedBy": String, "comment": String }` | `200 { "approvalId": String, "approved": false }` | `UpgradeJobEndpoint` → `ApprovalEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `UpgradeJobEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `UpgradeJobEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `UpgradeJobEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON shapes

### UpgradeJob (full form, returned by `GET /api/jobs/{id}`)

```json
{
  "jobId": "j-4a1…",
  "sourceVersion": "2.7.3",
  "targetVersion": "2.10.0",
  "requestedBy": "engineer@example.com",
  "status": "AWAITING_APPROVAL",
  "ledger": {
    "sourceVersion": "2.7.3",
    "targetVersion": "2.10.0",
    "phases": [
      { "phaseId": "compat-check", "name": "Compatibility scan", "kind": "COMPATIBILITY_CHECK", "requiresApproval": false },
      { "phaseId": "db-migrate",   "name": "Database schema migration", "kind": "MIGRATION", "requiresApproval": true },
      { "phaseId": "provider-update", "name": "Provider package update", "kind": "MIGRATION", "requiresApproval": true },
      { "phaseId": "final-tests",  "name": "Full test run", "kind": "TEST_RUN", "requiresApproval": false }
    ],
    "currentPhaseIndex": 1,
    "replanReason": null
  },
  "progress": {
    "entries": [
      {
        "attempt": 1,
        "executor": "COMPAT_CHECKER",
        "phaseId": "compat-check",
        "phaseName": "Compatibility scan",
        "verdict": "OK",
        "summary": "All 47 DAGs are compatible with 2.10.0. The google-cloud provider requires upgrade to 10.x.",
        "testReport": null,
        "blocker": null,
        "recordedAt": "2026-06-28T09:01:12Z"
      }
    ]
  },
  "pendingApproval": {
    "approvalId": "apr-9b3…",
    "jobId": "j-4a1…",
    "phaseId": "db-migrate",
    "phaseName": "Database schema migration",
    "rationale": "This phase runs alembic migrations against the Airflow metadata database. Schema changes are irreversible without the rollback script.",
    "requestedAt": "2026-06-28T09:01:45Z"
  },
  "report": null,
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T09:00:58Z",
  "finishedAt": null
}
```

### UpgradeJob (list form, returned by `GET /api/jobs`)

`UpgradeJobRow` mirrors `UpgradeJob` but `progress.entries` is truncated to the last 3 entries plus `truncatedFromTotal: int`. Each `summary` in a row entry is capped at 200 characters. The UI fetches the full job by id on click.

### ApprovalRequest (returned by `GET /api/approvals/{approvalId}`)

```json
{
  "approvalId": "apr-9b3…",
  "jobId": "j-4a1…",
  "phaseId": "db-migrate",
  "phaseName": "Database schema migration",
  "rationale": "This phase runs alembic migrations against the Airflow metadata database.",
  "requestedAt": "2026-06-28T09:01:45Z"
}
```

### SSE event format

```
event: job-update
data: { "jobId": "j-4a1…", "status": "AWAITING_APPROVAL", ... }
```

One event per state transition. Clients reconcile by `jobId`. The SSE channel also emits `event: control-update` when the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating unexpected CI failure", "haltedAt": "2026-06-28T09:15:00Z" }
```

And `event: approval-update` when a new approval request is created or resolved:

```
event: approval-update
data: { "approvalId": "apr-9b3…", "jobId": "j-4a1…", "phaseId": "db-migrate", "resolved": false }
```
