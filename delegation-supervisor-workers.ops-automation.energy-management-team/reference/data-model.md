# Data model

Every record the generated system defines. `Optional<T>` marks fields that are null before their lifecycle transition (Lesson 6). Akka's Jackson config serialises `Optional<T>` transparently — the wire value is the raw value or `null`.

## OptimizationCycle (entity state + view row)

| Field | Type | Nullable | Meaning |
|---|---|---|---|
| `cycleId` | `String` | no | UUID, also the workflow id |
| `buildingId` | `String` | no | Building identifier from the telemetry event |
| `zone` | `String` | no | Zone identifier within the building |
| `status` | `CycleStatus` | no | Lifecycle state |
| `hvacReport` | `Optional<HvacReport>` | yes | HvacSpecialist output; null until `HvacReportAttached` |
| `lightingReport` | `Optional<LightingReport>` | yes | LightingSpecialist output; null until `LightingReportAttached` |
| `equipmentReport` | `Optional<EquipmentReport>` | yes | EquipmentSpecialist output; null until `EquipmentReportAttached` |
| `consolidated` | `Optional<ConsolidatedReport>` | yes | Merged report; null until `CycleConsolidated` |
| `failureReason` | `Optional<String>` | yes | Set on `CyclePartial` or `CycleBlocked` |
| `evalScore` | `Optional<Integer>` | yes | 1–5; null until `EvalScored` |
| `evalRationale` | `Optional<String>` | yes | One-line eval justification |
| `startedAt` | `Instant` | no | Cycle creation time |
| `finishedAt` | `Optional<Instant>` | yes | Set on any terminal transition |

The `CycleView` row type mirrors this with `Optional<T>` on every nullable field.

## Supporting records

```java
record TelemetryEvent(String buildingId, String zone, Instant recordedAt) {}

record OptimizationPlan(String hvacQuery, String lightingQuery, String equipmentQuery) {}

record SetpointRecommendation(
    String unit,
    String parameter,
    double currentValue,
    double recommendedValue,
    String rationale
) {}

record HvacReport(
    List<SetpointRecommendation> recommendations,
    double currentLoadKw,
    Instant analysedAt
) {}

record LightingScheduleEntry(
    String zone,
    String startTime,
    String endTime,
    int targetLuxLevel,
    String rationale
) {}

record LightingReport(
    List<LightingScheduleEntry> scheduleChanges,
    double currentLoadKw,
    Instant analysedAt
) {}

record EquipmentAction(
    String assetId,
    String actionType,   // LOAD_SHIFT | SETBACK | SHUTDOWN_STANDBY
    String rationale,
    boolean blockedByGuardrail
) {}

record EquipmentReport(
    List<EquipmentAction> actions,
    double currentLoadKw,
    Instant analysedAt
) {}

record ConsolidatedReport(
    String summary,
    HvacReport hvacRecommendations,
    LightingReport lightingRecommendations,
    EquipmentReport equipmentRecommendations,
    double estimatedSavingsKwh,
    String guardrailVerdict,
    Instant consolidatedAt
) {}
```

## Status enum

```java
enum CycleStatus { QUEUED, DISPATCHED, CONSOLIDATING, COMPLETE, PARTIAL, BLOCKED }
```

## Events

### OptimizationCycleEntity

| Event | Trigger |
|---|---|
| `CycleStarted` | Workflow creates the cycle (`startCycle`) |
| `HvacReportAttached` | HvacSpecialist returns an `HvacReport` |
| `LightingReportAttached` | LightingSpecialist returns a `LightingReport` |
| `EquipmentReportAttached` | EquipmentSpecialist returns an `EquipmentReport` |
| `CycleConsolidated` | Coordinator consolidation completes; cycle moves to `COMPLETE` |
| `CyclePartial` | At least one specialist timed out; consolidation proceeded from partial input |
| `CycleBlocked` | Consolidation step failed a guardrail check |
| `EvalScored` | `EfficiencyEvalSampler` recorded a 1–5 quality score |

### TelemetryQueue

| Event | Trigger |
|---|---|
| `TelemetryReceived` | `receiveTelemetry(buildingId, zone)` from endpoint or simulator |

Fields: `{ cycleId, buildingId, zone, recordedAt }`.
