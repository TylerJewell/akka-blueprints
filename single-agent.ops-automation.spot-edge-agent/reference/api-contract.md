# API contract — spot-edge-agent

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| `POST` | `/api/missions` | `SubmitMissionRequest` | `201 { missionId }` | `MissionEndpoint` → `MissionEntity` |
| `GET` | `/api/missions` | — | `200 [ Mission... ]` (newest-first) | `MissionEndpoint` ← `MissionView` |
| `GET` | `/api/missions/{id}` | — | `200 Mission` / `404` | `MissionEndpoint` ← `MissionView` |
| `POST` | `/api/missions/{id}/approve` | — | `200 { missionId, status: "EXECUTING" }` / `409` | `MissionEndpoint` → `MissionEntity` |
| `POST` | `/api/missions/{id}/reject` | — | `200 { missionId, status: "FAILED" }` / `409` | `MissionEndpoint` → `MissionEntity` |
| `GET` | `/api/missions/sse` | — | `text/event-stream` | `MissionEndpoint` ← `MissionView` |
| `GET` | `/api/metadata/readme` | — | `text/markdown` | `AppEndpoint` |
| `GET` | `/api/metadata/risk-survey` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/api/metadata/eval-matrix` | — | `text/yaml` | `AppEndpoint` |
| `GET` | `/` | — | `302 → /app/index.html` | `AppEndpoint` |
| `GET` | `/app/*` | — | static UI | `AppEndpoint` |

## JSON payloads

### SubmitMissionRequest (request body)

```json
{
  "missionName": "Building-C overnight patrol",
  "route": [
    {
      "waypointId": "wp-entrance",
      "label": "Main entrance",
      "lat": 37.4219,
      "lng": -122.0840,
      "terrainType": "flat"
    },
    {
      "waypointId": "wp-server-room",
      "label": "Server room door",
      "lat": 37.4220,
      "lng": -122.0841,
      "terrainType": "flat"
    },
    {
      "waypointId": "wp-roof-stairs",
      "label": "Roof stairwell landing",
      "lat": 37.4221,
      "lng": -122.0842,
      "terrainType": "stair"
    }
  ],
  "payloadInstructions": "Capture thermal image at each waypoint.",
  "operatorCallsign": "ops-supervisor-7"
}
```

### Mission (response body)

```json
{
  "missionId": "m-4a3...",
  "brief": {
    "missionId": "m-4a3...",
    "missionName": "Building-C overnight patrol",
    "route": [ { "waypointId": "wp-entrance", "label": "Main entrance", "lat": 37.4219, "lng": -122.0840, "terrainType": "flat" } ],
    "payloadInstructions": "Capture thermal image at each waypoint.",
    "operatorCallsign": "ops-supervisor-7",
    "requiresApproval": true,
    "submittedAt": "2026-06-28T22:00:00Z"
  },
  "approvalSummary": {
    "proposedCommands": [
      {
        "commandId": "cmd-001",
        "spotCommand": "spot.change_gait",
        "params": { "gait": "stair" },
        "rationale": "Waypoint wp-roof-stairs has terrainType=stair; gait change required before navigation."
      },
      {
        "commandId": "cmd-002",
        "spotCommand": "spot.navigate_to",
        "params": { "waypointId": "wp-roof-stairs", "velocityMs": 0.3 },
        "rationale": "Navigate to stairwell landing at stair-terrain velocity cap."
      }
    ],
    "plainEnglishSummary": "The mission includes a stair-climb to the roof stairwell landing. The robot will switch to stair gait and navigate at 0.3 m/s. This maneuver requires operator approval.",
    "maneuverClass": "NON_TRIVIAL"
  },
  "approvalDecision": "APPROVED",
  "lastTelemetry": {
    "batteryPct": 82.4,
    "jointLoadMax": 0.31,
    "obstacleProximityM": 4.2,
    "estopSignal": false,
    "capturedAt": "2026-06-28T22:05:33Z"
  },
  "outcome": {
    "status": "COMPLETED",
    "waypointsReached": ["wp-entrance", "wp-server-room", "wp-roof-stairs"],
    "anomaliesObserved": [],
    "distanceTravelledM": 112.4,
    "completedAt": "2026-06-28T22:08:11Z"
  },
  "haltReason": null,
  "status": "COMPLETED",
  "createdAt": "2026-06-28T22:00:00Z",
  "finishedAt": "2026-06-28T22:08:11Z"
}
```

Any lifecycle field that has not happened yet is `null` on the wire (Akka serialises `Optional.empty()` as JSON `null`, not as `{}` or `{ "value": null }`).

### HaltReason (when status is HALTED)

```json
{
  "cause": "low-battery",
  "snapshot": {
    "batteryPct": 10.1,
    "jointLoadMax": 0.44,
    "obstacleProximityM": 3.8,
    "estopSignal": false,
    "capturedAt": "2026-06-28T22:06:44Z"
  },
  "lastCommand": "spot.navigate_to(waypointId=wp-server-room, velocityMs=1.4)",
  "haltedAt": "2026-06-28T22:06:44Z"
}
```

### SSE event format

```
event: mission-update
data: { "missionId": "m-4a3...", "status": "EXECUTING", "lastTelemetry": { ... }, ... }
```

One event per state transition (`SUBMITTED`, `PLANNING`, `PENDING_APPROVAL`, `EXECUTING`, `COMPLETED`, `HALTED`, `FAILED`). Clients reconcile by `missionId`; an event always carries the full row at the moment of transition, so a late-joining client never needs to replay.

## Authorization

ACL: open to the internet (local-dev only). A deployer adding identity must wrap the endpoint with their auth middleware and set `operatorCallsign` from the authenticated principal rather than the request body. The `approve` and `reject` endpoints should additionally check that the authenticated principal has the `robot-operator` role before forwarding to the entity.
