# API contract — nexshift

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/schedules` | `SubmitScheduleRequest` | `201 { scheduleId }` | `ScheduleEndpoint` → `ScheduleEntity` |
| `GET` | `/api/schedules` | — | `200 [ Schedule... ]` (newest-first) | `ScheduleEndpoint` ← `ScheduleView` |
| `GET` | `/api/schedules/{id}` | — | `200 Schedule` / `404` | `ScheduleEndpoint` ← `ScheduleView` |
| `PUT` | `/api/schedules/{id}/confirm` | — | `200 { scheduleId, status: "CONFIRMED" }` / `409` | `ScheduleEndpoint` → `ScheduleEntity` |
| `GET` | `/api/schedules/sse` | — | `text/event-stream` | `ScheduleEndpoint` ← `ScheduleView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitScheduleRequest (request body)

```json
{
  "weekStart": "2026-06-29",
  "department": "Nursing",
  "submittedBy": "manager-42",
  "openShifts": [
    {
      "shiftId": "shift-001",
      "role": "Nurse",
      "startTime": "2026-06-29T07:00:00Z",
      "endTime": "2026-06-29T15:00:00Z",
      "department": "Nursing",
      "requiredQualifications": ["cert-BLS"]
    },
    {
      "shiftId": "shift-002",
      "role": "Nurse",
      "startTime": "2026-06-29T15:00:00Z",
      "endTime": "2026-06-29T23:00:00Z",
      "department": "Nursing",
      "requiredQualifications": ["cert-BLS", "cert-ACLS"]
    }
  ],
  "employees": [
    {
      "employeeId": "emp-101",
      "name": "Jordan R.",
      "qualifications": ["cert-BLS", "cert-ACLS"],
      "availability": [
        { "day": "MONDAY", "from": "06:00", "to": "16:00" }
      ],
      "weeklyHoursCap": 40,
      "hoursScheduledThisWeek": 32
    },
    {
      "employeeId": "emp-102",
      "name": "Sam T.",
      "qualifications": ["cert-BLS"],
      "availability": [
        { "day": "MONDAY", "from": "14:00", "to": "00:00" }
      ],
      "weeklyHoursCap": 40,
      "hoursScheduledThisWeek": 16
    }
  ]
}
```

### Schedule (response body)

```json
{
  "scheduleId": "s-7af...",
  "request": {
    "scheduleId": "s-7af...",
    "weekStart": "2026-06-29",
    "department": "Nursing",
    "submittedBy": "manager-42",
    "submittedAt": "2026-06-28T09:00:00Z",
    "openShifts": [ "..." ],
    "employees": [ "..." ]
  },
  "draft": {
    "assignments": [
      {
        "shiftId": "shift-001",
        "employeeId": "emp-101",
        "status": "ASSIGNED",
        "blockedReason": null
      },
      {
        "shiftId": "shift-002",
        "employeeId": "emp-102",
        "status": "ASSIGNED",
        "blockedReason": null
      }
    ],
    "totalShifts": 2,
    "filledCount": 2,
    "blockedCount": 0
  },
  "status": "CONFIRMED",
  "createdAt": "2026-06-28T09:00:00Z",
  "confirmedAt": "2026-06-28T09:05:12Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: schedule-update
data: { "scheduleId": "s-7af...", "status": "DRAFT", "draft": { ... }, ... }
```

One event per state transition (`PENDING`, `BUILDING`, `SCHEDULING`, `DRAFT`, `CONFIRMED`, `FAILED`). Clients reconcile by `scheduleId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body. The confirm endpoint should additionally enforce that only the submitting manager (or a designated approver role) can confirm.
