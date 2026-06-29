# API contract — field-service-dispatcher

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/schedules` | `{ "description": String, "serviceAddress": String, "requiredSkill": String, "requestedBy"?: String }` | `202 { "scheduleId": String }` | `ScheduleEndpoint` → `WorkOrderQueue` |
| `GET` | `/api/schedules` | — | `200 [ ScheduleRow... ]` | `ScheduleEndpoint` ← `ScheduleView` |
| `GET` | `/api/schedules/{id}` | — | `200 Schedule` / `404` | `ScheduleEndpoint` ← `ScheduleEntity` |
| `GET` | `/api/schedules/sse` | — | `text/event-stream` (one event per schedule change) | `ScheduleEndpoint` ← `ScheduleView` |
| `POST` | `/api/control/pause` | `{ "reason": String }` | `200 { "paused": true, "reason": String }` | `ScheduleEndpoint` → `OperatorControlEntity` |
| `POST` | `/api/control/resume` | — | `200 { "paused": false }` | `ScheduleEndpoint` → `OperatorControlEntity` |
| `GET` | `/api/control` | — | `200 { "paused": boolean, "reason"?: String, "pausedAt"?: Instant }` | `ScheduleEndpoint` ← `OperatorControlEntity` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI (single self-contained `index.html`) | `AppEndpoint` |

## JSON shapes

### Schedule (full form, returned by `GET /api/schedules/{id}`)

```json
{
  "scheduleId": "s-9a1…",
  "description": "Assign next plumbing work order to Zone 3 technician",
  "serviceAddress": "14 Oak St, Zone 3",
  "requiredSkill": "plumbing",
  "status": "DISPATCHING",
  "ledger": {
    "knownOrders": ["WO-221: plumbing repair at 14 Oak St, Zone 3"],
    "unassignedSlots": ["WO-221"],
    "plan": [
      "Check availability of Zone 3 plumbing-qualified technicians",
      "Route-optimize to assign closest technician to WO-221",
      "Verify technician is within shift window"
    ],
    "currentDispatch": {
      "specialist": "AVAILABILITY",
      "assignment": "Verify T-07 is on-shift for WO-221",
      "rationale": "Confirming shift window before committing the assignment."
    }
  },
  "fairness": {
    "entries": [
      {
        "attempt": 1,
        "specialist": "AVAILABILITY",
        "assignment": "Verify T-07 is on-shift for WO-221",
        "verdict": "OK",
        "scrubbedResult": "T-07 on shift until 17:00, 2 of 4 assignments used.",
        "blocker": null,
        "recordedAt": "2026-06-28T09:14:02Z"
      }
    ],
    "alerts": []
  },
  "summary": null,
  "failureReason": null,
  "pauseReason": null,
  "createdAt": "2026-06-28T09:13:55Z",
  "finishedAt": null
}
```

### Schedule (list form, returned by `GET /api/schedules`)

The list form is `ScheduleRow` — same fields as `Schedule`, but `fairness.entries` is truncated to the last 3 entries plus `truncatedFromTotal: <int>` and each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full schedule by id on row expand.

### SSE event format

```
event: schedule-update
data: { "scheduleId": "s-9a1…", "status": "DISPATCHING", ... }
```

One event per state transition. Clients reconcile by `scheduleId`. The SSE channel also emits `event: control-update` whenever the operator pause flag changes:

```
event: control-update
data: { "paused": true, "reason": "traffic incident in Zone 3", "pausedAt": "2026-06-28T09:20:11Z" }
```

And `event: fairness-alert` when a `FairnessAlertRecorded` event fires:

```
event: fairness-alert
data: { "scheduleId": "s-9a1…", "technicianId": "T-02", "alertType": "overloaded", "detail": "T-02 has 62% of today's assignments vs 14% fleet average" }
```
