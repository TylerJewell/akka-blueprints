# API contract — computer-use-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/tasks` | `SubmitTaskRequest` | `201 { taskId }` | `TaskEndpoint` → `TaskEntity` |
| `GET` | `/api/tasks` | — | `200 [ TaskRow... ]` (newest-first) | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/tasks/{id}` | — | `200 TaskRow` / `404` | `TaskEndpoint` ← `TaskView` |
| `POST` | `/api/tasks/{id}/halt` | — | `204` | `TaskEndpoint` → `TaskEntity` |
| `POST` | `/api/tasks/{id}/confirm` | `ConfirmationResponse` | `204` | `TaskEndpoint` → `TaskEntity` |
| `GET` | `/api/tasks/sse` | — | `text/event-stream` | `TaskEndpoint` ← `TaskView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitTaskRequest (POST /api/tasks body)

```json
{
  "description": "Open the contact form at https://forms.example.com/contact, fill in name 'Jane Smith', email 'jane@example.com', subject 'Partnership inquiry', then submit.",
  "templateId": "fill-web-form",
  "submittedBy": "ops-user-42"
}
```

`templateId` is optional. When present, `EnvironmentSimulator` seeds the starting screenshot from the matching template. When absent, the generic placeholder screenshot is used.

### TaskRow (response body for GET /api/tasks/{id})

```json
{
  "taskId": "t-9ae...",
  "request": {
    "taskId": "t-9ae...",
    "description": "Open the contact form at https://forms.example.com/contact ...",
    "templateId": "fill-web-form",
    "submittedAt": "2026-06-28T14:20:00Z",
    "submittedBy": "ops-user-42"
  },
  "actionHistory": [
    {
      "callId": "a1b2c3d4-...",
      "actionType": "NAVIGATE",
      "targetSelector": "https://forms.example.com/contact",
      "payload": "",
      "status": "ALLOWED",
      "blockedReason": null,
      "executedAt": "2026-06-28T14:20:01Z"
    },
    {
      "callId": "e5f6a7b8-...",
      "actionType": "TYPE",
      "targetSelector": "#first-name",
      "payload": "Jane",
      "status": "ALLOWED",
      "blockedReason": null,
      "executedAt": "2026-06-28T14:20:02Z"
    },
    {
      "callId": "c9d0e1f2-...",
      "actionType": "FORM_SUBMIT",
      "targetSelector": "#contact-form",
      "payload": "",
      "status": "CONFIRMED",
      "blockedReason": null,
      "executedAt": "2026-06-28T14:21:55Z"
    }
  ],
  "pendingConfirmation": null,
  "outcome": {
    "status": "SUCCESS",
    "summary": "Filled the contact form with the provided values and confirmed submission. The confirmation page displayed 'Form received — reference #4892'.",
    "totalActionsExecuted": 7,
    "actionsBlocked": 0,
    "actionsConfirmed": 1,
    "completedAt": "2026-06-28T14:21:56Z"
  },
  "status": "COMPLETED",
  "haltRequested": false,
  "createdAt": "2026-06-28T14:20:00Z",
  "finishedAt": "2026-06-28T14:21:56Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (`Optional.empty()` serialises as JSON `null`, not as `{}` or `{ "value": null }`).

### ConfirmationResponse (POST /api/tasks/{id}/confirm body)

```json
{
  "confirmationId": "conf-3b7a...",
  "approved": true,
  "respondedBy": "ops-user-42"
}
```

### Blocked ActionRecord example

When the guardrail blocks an action, the `ActionRecord` written by the workflow has `status = BLOCKED` and a non-null `blockedReason`:

```json
{
  "callId": "f3a4b5c6-...",
  "actionType": "FILE_DELETE",
  "targetSelector": "/data/reports/q2-2026.csv",
  "payload": "",
  "status": "BLOCKED",
  "blockedReason": "BLOCKED_DESTRUCTIVE_ACTION: FILE_DELETE requires a prior ConfirmationReceived event in this task's action history.",
  "executedAt": "2026-06-28T14:22:10Z"
}
```

### SSE event format

```
event: task-update
data: { "taskId": "t-9ae...", "status": "AWAITING_CONFIRMATION", "pendingConfirmation": { "confirmationId": "conf-3b7a...", "pendingAction": { ... }, "riskRationale": "FORM_SUBMIT actions trigger external workflows and cannot be automatically undone.", "requestedAt": "2026-06-28T14:21:30Z" }, ... }
```

One event per state transition (`SUBMITTED`, `RUNNING`, `AWAITING_CONFIRMATION`, `COMPLETED`, `HALTED`, `FAILED`). Each event carries the full `TaskRow` at the moment of transition. An `action-executed` sub-event fires on every `ActionExecuted` entity event to update the action log in the UI without a full state transition.

```
event: action-executed
data: { "taskId": "t-9ae...", "action": { "callId": "e5f6a7b8-...", "actionType": "TYPE", ... } }
```

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and populate `submittedBy` from the authenticated principal. The halt and confirm endpoints should be restricted to the submitting user or a designated operator role.
