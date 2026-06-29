# API contract

All endpoints are JSON unless noted. ACL: open to the internet (local-dev only). Base path `/api`.

| Method | Path | Body | Response | Component |
|---|---|---|---|---|
| POST | `/api/cycles` | `{ "buildingId": "string", "zone": "string" }` | `{ "cycleId": "uuid" }` | `CycleEndpoint` → `TelemetryQueue` |
| GET | `/api/cycles` | — | `{ "cycles": [OptimizationCycleRow, ...] }` | `CycleEndpoint` → `CycleView` |
| GET | `/api/cycles?status=COMPLETE` | — | filtered list (client-side filter) | `CycleEndpoint` |
| GET | `/api/cycles/{id}` | — | `OptimizationCycleRow` or 404 | `CycleEndpoint` |
| GET | `/api/cycles/sse` | — | `text/event-stream` of `OptimizationCycleRow` | `CycleEndpoint` → `CycleView` |
| GET | `/api/metadata/readme` | — | `text/markdown` | `CycleEndpoint` |
| GET | `/api/metadata/risk-survey` | — | `text/yaml` | `CycleEndpoint` |
| GET | `/api/metadata/eval-matrix` | — | `text/yaml` | `CycleEndpoint` |
| GET | `/` | — | 302 → `/app/index.html` | `AppEndpoint` |
| GET | `/app/{*path}` | — | static resource | `AppEndpoint` |

## Payload shapes

`POST /api/cycles` request:

```json
{ "buildingId": "HQ-WEST", "zone": "FLOOR-3-SOUTH" }
```

`OptimizationCycleRow` (lifecycle fields are `Optional` in Java; null on the wire until their transition):

```json
{
  "cycleId": "uuid",
  "buildingId": "string",
  "zone": "string",
  "status": "QUEUED | DISPATCHED | CONSOLIDATING | COMPLETE | PARTIAL | BLOCKED",
  "hvacReport": {
    "recommendations": [
      { "unit": "AHU-03", "parameter": "coolingSetpoint", "currentValue": 22.0,
        "recommendedValue": 24.0, "rationale": "Zone at 28% occupancy; relaxing by 2°C reduces compressor runtime." }
    ],
    "currentLoadKw": 14.2,
    "analysedAt": "ISO-8601"
  },
  "lightingReport": {
    "scheduleChanges": [
      { "zone": "FLOOR-3-SOUTH-PERIMETER", "startTime": "09:00", "endTime": "15:00",
        "targetLuxLevel": 150, "rationale": "Daylight exceeds 500 lux; artificial lighting can step down." }
    ],
    "currentLoadKw": 3.6,
    "analysedAt": "ISO-8601"
  },
  "equipmentReport": {
    "actions": [
      { "assetId": "UPS-RACK-07", "actionType": "SHUTDOWN_STANDBY",
        "rationale": "Drawing 4.2 kW at 18% load outside business hours.",
        "blockedByGuardrail": false }
    ],
    "currentLoadKw": 8.1,
    "analysedAt": "ISO-8601"
  },
  "consolidated": {
    "summary": "string (80–120 words)",
    "estimatedSavingsKwh": 12.4,
    "guardrailVerdict": "ok",
    "consolidatedAt": "ISO-8601"
  },
  "failureReason": "string or null",
  "evalScore": "1-5 or null",
  "evalRationale": "string or null",
  "startedAt": "ISO-8601",
  "finishedAt": "ISO-8601 or null"
}
```

## SSE event format

`GET /api/cycles/sse` emits one event per cycle change:

```
event: cycle
data: { ...OptimizationCycleRow... }

```

The UI's App UI tab subscribes via `EventSource` and upserts each row into its live list by `cycleId`. No polling.
