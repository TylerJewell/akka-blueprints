# API contract — itsm-change-planner

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/changes` | `{ "summary": String, "ciName": String, "requestedBy"?: String, "category"?: String }` | `202 { "changeId": String }` | `ChangeEndpoint` → `ChangeQueue` |
| `GET` | `/api/changes` | — | `200 [ ChangeRow... ]` | `ChangeEndpoint` ← `ChangeView` |
| `GET` | `/api/changes/{id}` | — | `200 Change` / `404` | `ChangeEndpoint` ← `ChangeEntity` |
| `GET` | `/api/changes/sse` | — | `text/event-stream` (one event per change state transition) | `ChangeEndpoint` ← `ChangeView` |
| `POST` | `/api/changes/{id}/approve` | `{ "reviewedBy": String, "comments"?: String }` | `200 { "changeId": String, "status": "APPROVED" }` | `ChangeEndpoint` → `ChangeEntity` |
| `POST` | `/api/changes/{id}/reject` | `{ "reviewedBy": String, "comments"?: String }` | `200 { "changeId": String, "status": "REJECTED" }` | `ChangeEndpoint` → `ChangeEntity` |
| `POST` | `/api/control/halt` | `{ "reason": String }` | `200 { "halted": true, "reason": String }` | `ChangeEndpoint` → `SystemControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "halted": false }` | `ChangeEndpoint` → `SystemControlEntity` |
| `GET` | `/api/control` | — | `200 { "halted": boolean, "reason"?: String, "haltedAt"?: Instant }` | `ChangeEndpoint` ← `SystemControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Change (full form, returned by `GET /api/changes/{id}`)

```json
{
  "changeId": "chg-4a3…",
  "summary": "Upgrade nginx to 1.26.1 on web-prod-01",
  "ciName": "web-prod-01",
  "category": "NORMAL",
  "requestedBy": "ops-team",
  "status": "EXECUTING",
  "ledger": {
    "impactAssessment": "web-prod-01 serves public HTTPS traffic. Upgrading nginx may cause a brief connection reset. Related CI: lb-prod-01 (load balancer).",
    "similarChanges": [
      {
        "changeId": "chg-001",
        "summary": "Upgraded nginx on web-staging-01",
        "outcome": "succeeded",
        "lessonsLearned": "Reload with nginx -s reload avoids a full restart."
      }
    ],
    "implementationPlan": [
      {
        "sequence": 1,
        "description": "Download nginx 1.26.1 package to web-prod-01.",
        "targetCi": "web-prod-01",
        "expectedOutcome": "Package present at /var/cache/apt."
      },
      {
        "sequence": 2,
        "description": "Install nginx 1.26.1 on web-prod-01.",
        "targetCi": "web-prod-01",
        "expectedOutcome": "nginx --version reports 1.26.1."
      }
    ],
    "testPlan": [
      {
        "sequence": 1,
        "description": "Verify package file present.",
        "successCriteria": "ls /var/cache/apt shows nginx_1.26.1 package."
      },
      {
        "sequence": 2,
        "description": "Verify nginx reports new version.",
        "successCriteria": "nginx --version output contains 1.26.1."
      }
    ],
    "backoutPlan": [
      {
        "sequence": 1,
        "description": "Downgrade nginx to previous version on web-prod-01.",
        "targetCi": "web-prod-01"
      }
    ]
  },
  "executionLog": {
    "records": [
      {
        "sequence": 1,
        "description": "Download nginx 1.26.1 package to web-prod-01.",
        "stepResult": {
          "sequence": 1,
          "description": "Download nginx 1.26.1 package to web-prod-01.",
          "ok": true,
          "evidence": "apt-get download nginx=1.26.1 executed on web-prod-01.\nPackage cached at /var/cache/apt/archives/nginx_1.26.1_amd64.deb.\nChecksum verified: SHA256 matches upstream.",
          "errorReason": null
        },
        "testResult": {
          "sequence": 1,
          "passed": true,
          "observation": "ls /var/cache/apt/archives shows nginx_1.26.1_amd64.deb present. File size 450 KB matches expected.",
          "failureDetail": null
        },
        "recordedAt": "2026-06-28T11:05:03Z"
      }
    ]
  },
  "cabDecision": {
    "outcome": "APPROVED",
    "reviewedBy": "j.smith@example.com",
    "comments": "Approved for off-peak window.",
    "decidedAt": "2026-06-28T11:02:44Z"
  },
  "failureReason": null,
  "haltReason": null,
  "createdAt": "2026-06-28T11:00:01Z",
  "finishedAt": null
}
```

### ChangeRow (list form, returned by `GET /api/changes`)

`ChangeRow` mirrors `Change` but omits `ledger.implementationPlan`, `ledger.testPlan`, and `ledger.backoutPlan`; `executionLog.records` is truncated to the last 3 entries plus `truncatedFromTotal: int`. The UI fetches the full change by id on row expand.

### SSE event format

```
event: change-update
data: { "changeId": "chg-4a3…", "status": "EXECUTING", ... }
```

One event per state transition. Clients reconcile by `changeId`. The SSE channel also emits `event: control-update` whenever the operator halt flag changes:

```
event: control-update
data: { "halted": true, "reason": "investigating failed test on web-prod-01", "haltedAt": "2026-06-28T11:07:15Z" }
```
