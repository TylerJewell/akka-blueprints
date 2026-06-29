# API contract — web-navigation-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/sessions` | `StartSessionRequest` | `201 { sessionId }` | `SessionEndpoint` → `SessionEntity` |
| `GET` | `/api/sessions` | — | `200 [ Session... ]` (newest-first) | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/{id}` | — | `200 Session` / `404` | `SessionEndpoint` ← `SessionView` |
| `GET` | `/api/sessions/sse` | — | `text/event-stream` | `SessionEndpoint` ← `SessionView` |
| `POST` | `/api/sessions/{id}/approve` | `HitlApprovalRequest` | `200` / `404` / `409` | `SessionEndpoint` → `NavigationWorkflow` |
| `POST` | `/api/sessions/{id}/reject` | `HitlRejectionRequest` | `200` / `404` / `409` | `SessionEndpoint` → `NavigationWorkflow` |
| `POST` | `/api/halt` | — | `200` | `SessionEndpoint` → `HaltController` |
| `DELETE` | `/api/halt` | — | `200` | `SessionEndpoint` → `HaltController` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### StartSessionRequest (request body)

```json
{
  "description": "Find the latest Akka SDK release version on doc.akka.io",
  "startingUrl": "https://doc.akka.io",
  "maxSteps": 20,
  "submittedBy": "qa-engineer-07"
}
```

### HitlApprovalRequest (request body)

```json
{
  "reviewedBy": "ops-lead-02"
}
```

### HitlRejectionRequest (request body)

```json
{
  "reviewedBy": "ops-lead-02",
  "reason": "Checkout step not authorised for this session."
}
```

### Session (response body)

```json
{
  "sessionId": "s-a3f...",
  "goal": {
    "goalId": "s-a3f...",
    "description": "Find the latest Akka SDK release version on doc.akka.io",
    "startingUrl": "https://doc.akka.io",
    "maxSteps": 20,
    "submittedBy": "qa-engineer-07",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "actionLog": [
    {
      "actionId": "act-001",
      "action": {
        "actionType": "NAVIGATE",
        "targetSelector": "",
        "targetUrl": "https://doc.akka.io",
        "inputText": "",
        "rationale": "Starting navigation at the requested URL.",
        "highStakes": false,
        "decidedAt": "2026-06-28T14:00:01Z"
      },
      "executed": true,
      "blockedReason": null,
      "screenshotPath": "sessions/s-a3f.../step-001.png",
      "executedAt": "2026-06-28T14:00:02Z"
    }
  ],
  "pendingAction": null,
  "lastHitlDecision": null,
  "outcome": {
    "success": true,
    "taskResult": "Latest Akka SDK release is 3.6.0, found at doc.akka.io/docs/akka/current/release-notes.html",
    "stepsUsed": 7,
    "completedAt": "2026-06-28T14:00:35Z"
  },
  "status": "COMPLETED",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": "2026-06-28T14:00:35Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### SSE event format

```
event: session-update
data: { "sessionId": "s-a3f...", "status": "NAVIGATING", "actionLog": [...], ... }
```

One event per state transition (`STARTING`, `NAVIGATING`, `AWAITING_APPROVAL`, `COMPLETED`, `HALTED`, `FAILED`, `REJECTED`) and per `ActionExecuted` / `ScreenshotCaptured` event. Clients reconcile by `sessionId`; an event always carries the full row at the moment of emission, so a late-joining client does not need to replay.

### HITL pending action in response

When `status = "AWAITING_APPROVAL"`, the `pendingAction` field is populated:

```json
{
  "pendingAction": {
    "actionType": "CLICK",
    "targetSelector": "button#confirm-purchase",
    "targetUrl": "",
    "inputText": "",
    "rationale": "The checkout page shows a confirm button; clicking it completes the purchase.",
    "highStakes": true,
    "decidedAt": "2026-06-28T14:01:00Z"
  }
}
```

The UI renders this in the right pane with Approve / Reject controls. The reviewer can read the `rationale` and inspect the `targetSelector` before deciding.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap `SessionEndpoint` with their auth middleware. The `POST /api/halt` endpoint in particular should be restricted to operators — any authenticated user reaching it can stop all running sessions.
