# API contract — self-improving-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `{ "instruction": String, "expectedHint"?: String, "submittedBy"?: String }` | `202 { "taskId": String }` | `AgentEndpoint` → `TaskQueueEntity` |
| `GET` | `/api/configs` | — | `200 [ AgentConfig... ]` (optional `?status=ACTIVE\|IMPROVING\|APPLIED\|REVISION_REJECTED` — filtered client-side from `getAllConfigs`) | `AgentEndpoint` ← `ConfigView` |
| `GET` | `/api/configs/{id}` | — | `200 AgentConfig` / `404` | `AgentEndpoint` ← `ConfigView` |
| `GET` | `/api/configs/sse` | — | `text/event-stream` (one event per config change) | `AgentEndpoint` ← `ConfigView` |
| `GET` | `/api/configs/monitor/sse` | — | `text/event-stream` (terminal transitions only: RevisionApplied, RevisionRejected) | `AgentEndpoint` ← `AgentConfigEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## Defaults on `POST /api/tasks`

- Missing `expectedHint` → `""` (empty string; informational only).
- Missing `submittedBy` → `"anonymous"`.
- `instruction` must be non-empty; otherwise `400`.
- Duplicate-detection window: 10 s on `(instruction, submittedBy)`; the second submission returns the first `taskId` (200) instead of enqueuing a duplicate.

## JSON shapes

### AgentConfig

```json
{
  "configId": "cfg-4a1b2…",
  "systemPrompt": "You are a task executor. Process the instruction provided…",
  "status": "APPLIED",
  "cycles": [
    {
      "cycleNumber": 1,
      "proposal": {
        "proposedSystemPrompt": "You are a task executor. Return results as bullet points…",
        "rationale": "Batch showed verbosity issues; bullet format reduces word count.",
        "revisionAttempt": 1
      },
      "attestation": {
        "passed": false,
        "scoreDelta": -0.1,
        "detail": "Regression score 3.4 vs. baseline 3.5; threshold 0.5 not met."
      },
      "appliedConfigId": null
    },
    {
      "cycleNumber": 2,
      "proposal": {
        "proposedSystemPrompt": "You are a task executor. Return results in prose, max 150 words…",
        "rationale": "Bullet format lost context on multi-step tasks; prose with ceiling restores coherence.",
        "revisionAttempt": 2
      },
      "attestation": {
        "passed": true,
        "scoreDelta": 0.8,
        "detail": "Regression score 4.3 vs. baseline 3.5; threshold met."
      },
      "appliedConfigId": "cfg-9c3f7…"
    }
  ],
  "parentConfigId": "cfg-0000…",
  "rejectionReason": null,
  "createdAt": "2026-06-28T09:00:00Z",
  "finalizedAt": "2026-06-28T09:04:11Z"
}
```

### AttestationVerdict form

```json
{ "passed": true,  "scoreDelta": 0.8,  "detail": "Regression score 4.3 vs. baseline 3.5; threshold met." }
{ "passed": false, "scoreDelta": -0.1, "detail": "Regression score 3.4 vs. baseline 3.5; threshold 0.5 not met." }
```

### SSE event format — general config stream

```
event: config-update
data: { "configId": "cfg-4a1b2…", "status": "APPLIED", "cycles": [...], ... }
```

One event per state transition. Clients reconcile by `configId`. The full `AgentConfig` JSON is included so a fresh client can render the row without a separate fetch.

### SSE event format — monitor stream

```
event: revision-applied
data: { "configId": "cfg-4a1b2…", "appliedAt": "2026-06-28T09:04:11Z", "cycleNumber": 2, "scoreDelta": 0.8 }

event: revision-rejected
data: { "configId": "cfg-4a1b2…", "rejectedAt": "2026-06-28T09:10:03Z", "totalCycles": 3, "rejectionReason": "max revisions reached (3)" }
```

Terminal-only; the monitor stream does not emit intermediate `IMPROVING` transitions.
