# API contract — personal-assistant

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/requests` | `SubmitAssistantRequest` | `201 { requestId }` | `AssistantEndpoint` → `AssistantEntity` |
| `GET` | `/api/requests` | — | `200 [ AssistantRow... ]` (newest-first) | `AssistantEndpoint` ← `AssistantView` |
| `GET` | `/api/requests/{id}` | — | `200 AssistantRow` / `404` | `AssistantEndpoint` ← `AssistantView` |
| `GET` | `/api/requests/sse` | — | `text/event-stream` | `AssistantEndpoint` ← `AssistantView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitAssistantRequest (request body)

```json
{
  "naturalLanguageRequest": "Schedule a hotel check-in reminder for Monday at 9 AM",
  "rawContext": {
    "events": [
      {
        "eventId": "evt-001",
        "title": "Flight LH 456 departure",
        "date": "2026-07-07",
        "time": "06:30",
        "location": "Frankfurt Airport Terminal 1"
      },
      {
        "eventId": "evt-002",
        "title": "Hotel check-in",
        "date": "2026-07-07",
        "time": null,
        "location": "123 Main St, [REDACTED-ADDRESS]"
      }
    ],
    "tasks": [
      {
        "taskId": "task-001",
        "title": "Pack sunscreen",
        "completed": false,
        "dueDate": "2026-07-06",
        "priority": "NORMAL"
      }
    ],
    "timezone": "Europe/Berlin"
  },
  "submittedBy": "user-7731"
}
```

### AssistantRow (response body)

```json
{
  "requestId": "req-3a9...",
  "request": {
    "requestId": "req-3a9...",
    "naturalLanguageRequest": "Schedule a hotel check-in reminder for Monday at 9 AM",
    "rawContext": "(raw context preserved for audit)",
    "submittedBy": "user-7731",
    "submittedAt": "2026-06-28T10:00:00Z"
  },
  "sanitized": {
    "redactedSnapshot": "{\"events\":[{\"eventId\":\"evt-001\",\"title\":\"Flight LH 456 departure\",\"date\":\"2026-07-07\",\"time\":\"06:30\",\"location\":\"Frankfurt Airport Terminal 1\"},{\"eventId\":\"evt-002\",\"title\":\"Hotel check-in\",\"date\":\"2026-07-07\",\"time\":null,\"location\":\"[REDACTED-ADDRESS]\"}], ...}",
    "piiCategoriesFound": ["address"]
  },
  "action": {
    "actionType": "SET_REMINDER",
    "explanation": "Added a reminder for the hotel check-in event on Monday at 9 AM.",
    "changeSet": [
      { "field": "targetId", "oldValue": null, "newValue": "evt-002" },
      { "field": "time", "oldValue": null, "newValue": "09:00" },
      { "field": "reminderMinutesBefore", "oldValue": null, "newValue": "60" }
    ],
    "decidedAt": "2026-06-28T10:00:18Z"
  },
  "eval": {
    "score": 5,
    "rationale": "Action targets an existing event, change set is specific and non-empty, action type matches the request intent.",
    "evaluatedAt": "2026-06-28T10:00:19Z"
  },
  "status": "EVALUATED",
  "createdAt": "2026-06-28T10:00:00Z",
  "finishedAt": "2026-06-28T10:00:19Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: request-update
data: { "requestId": "req-3a9...", "status": "ACTION_RECORDED", "action": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `SANITIZED`, `ACTING`, `ACTION_RECORDED`, `EVALUATED`, `FAILED`). Clients reconcile by `requestId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `submittedBy` from the authenticated principal rather than the request body.
