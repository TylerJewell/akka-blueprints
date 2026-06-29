# API contract — smart-home-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/commands` | `SubmitCommandRequest` | `201 { commandId }` | `CommandEndpoint` → `CommandEntity` |
| `GET` | `/api/commands` | — | `200 [ Command... ]` (newest-first) | `CommandEndpoint` ← `CommandView` |
| `GET` | `/api/commands/{id}` | — | `200 Command` / `404` | `CommandEndpoint` ← `CommandView` |
| `GET` | `/api/commands/sse` | — | `text/event-stream` | `CommandEndpoint` ← `CommandView` |
| `POST` | `/api/commands/{id}/confirm` | — | `200` / `409 if not AWAITING_CONFIRMATION` | `CommandEndpoint` → `CommandEntity` |
| `POST` | `/api/commands/{id}/cancel` | — | `200` / `409 if not AWAITING_CONFIRMATION` | `CommandEndpoint` → `CommandEntity` |
| `GET` | `/api/devices` | — | `200 [ Device... ]` | `CommandEndpoint` ← `DeviceRegistry` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitCommandRequest (request body)

```json
{
  "homeId": "home-1",
  "rawCommand": "Lock the front door",
  "submittedBy": "resident-alice"
}
```

### Command (response body — AWAITING_CONFIRMATION state)

```json
{
  "commandId": "cmd-3a9...",
  "request": {
    "commandId": "cmd-3a9...",
    "homeId": "home-1",
    "rawCommand": "Lock the front door",
    "submittedBy": "resident-alice",
    "submittedAt": "2026-06-28T14:00:00Z"
  },
  "action": {
    "deviceId": "front-door-lock",
    "actionType": "LOCK",
    "numericParam": null,
    "rationale": "Resident requested front door lock; mapped to LOCK action on front-door-lock."
  },
  "guardResult": {
    "passed": true,
    "rejectionReason": null
  },
  "status": "AWAITING_CONFIRMATION",
  "createdAt": "2026-06-28T14:00:00Z",
  "finishedAt": null
}
```

### Command (response body — BLOCKED state)

```json
{
  "commandId": "cmd-7f2...",
  "request": {
    "commandId": "cmd-7f2...",
    "homeId": "home-1",
    "rawCommand": "Set the thermostat to 95°F",
    "submittedBy": "resident-alice",
    "submittedAt": "2026-06-28T14:01:00Z"
  },
  "action": null,
  "guardResult": {
    "passed": false,
    "rejectionReason": "parameter-out-of-range: numericParam=95 is outside [60, 80] for device thermostat-main"
  },
  "status": "BLOCKED",
  "createdAt": "2026-06-28T14:01:00Z",
  "finishedAt": "2026-06-28T14:01:02Z"
}
```

Any lifecycle field that has not yet been set is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`).

### Device (response body element from GET /api/devices)

```json
{
  "deviceId": "kitchen-light",
  "displayName": "Kitchen Light",
  "deviceClass": "LIGHT",
  "location": "kitchen",
  "minValue": 0,
  "maxValue": 100
}
```

### SSE event format

```
event: command-update
data: { "commandId": "cmd-3a9...", "status": "DISPATCHED", "action": { ... }, ... }
```

One event per state transition (`RECEIVED`, `GUARD_PASSED`, `BLOCKED`, `AWAITING_CONFIRMATION`, `DISPATCHED`, `CANCELLED`, `FAILED`). Clients reconcile by `commandId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and populate `submittedBy` from the authenticated principal. The confirm/cancel endpoints should also validate that the calling identity matches the original `submittedBy`.
