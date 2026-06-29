# Data model — spot-edge-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Waypoint` | `waypointId` | `String` | no | Stable id used throughout the mission. |
| | `label` | `String` | no | Human-readable location name. |
| | `lat` | `double` | no | WGS-84 latitude. |
| | `lng` | `double` | no | WGS-84 longitude. |
| | `terrainType` | `String` | no | One of: `flat`, `mixed`, `stair`. Governs gait and velocity cap. |
| `MissionBrief` | `missionId` | `String` | no | UUID minted by `MissionEndpoint`. |
| | `missionName` | `String` | no | Operator-supplied label. |
| | `route` | `List<Waypoint>` | no | Ordered list of waypoints (1–N). |
| | `payloadInstructions` | `String` | no | Free-text instruction for the agent (e.g., "capture thermal image at each waypoint"). |
| | `operatorCallsign` | `String` | no | Operator identifier. |
| | `requiresApproval` | `boolean` | no | Set true during planning if any command is `NON_TRIVIAL`. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `CommandProposal` | `commandId` | `String` | no | Stable id minted during planning. |
| | `spotCommand` | `String` | no | Tool name: `spot.navigate_to`, `spot.capture_image`, etc. |
| | `params` | `Map<String, Object>` | no | Command parameters. |
| | `rationale` | `String` | no | Agent's reason for the command (shown in the approval panel). |
| `ApprovalSummary` | `proposedCommands` | `List<CommandProposal>` | no | Full command sequence for the operator's review. |
| | `plainEnglishSummary` | `String` | no | 1–3 sentences describing the non-trivial aspect. |
| | `maneuverClass` | `ManeuverClass` | no | Enum value. |
| `TelemetrySnapshot` | `batteryPct` | `double` | no | Battery charge percentage (0–100). |
| | `jointLoadMax` | `double` | no | Maximum joint load across all joints (0.0–1.0). |
| | `obstacleProximityM` | `double` | no | Distance to nearest detected obstacle in metres. |
| | `estopSignal` | `boolean` | no | True if the hardware E-stop is asserted. |
| | `capturedAt` | `Instant` | no | When the reading was taken. |
| `MissionOutcome` | `status` | `OutcomeStatus` | no | Enum value. |
| | `waypointsReached` | `List<String>` | no | Waypoint IDs successfully reached. |
| | `anomaliesObserved` | `List<String>` | no | Human-readable anomaly descriptions. |
| | `distanceTravelledM` | `double` | no | Total distance covered. |
| | `completedAt` | `Instant` | no | When the agent returned the outcome. |
| `HaltReason` | `cause` | `String` | no | One of: `low-battery`, `joint-overload`, `obstacle-proximity`, `estop`. |
| | `snapshot` | `TelemetrySnapshot` | no | Sensor readings at the moment of halt. |
| | `lastCommand` | `String` | no | String representation of the last agent-issued command. |
| | `haltedAt` | `Instant` | no | When the halt fired. |
| `Mission` (entity state) | `missionId` | `String` | no | — |
| | `brief` | `Optional<MissionBrief>` | yes | Populated after `MissionSubmitted`. |
| | `approvalSummary` | `Optional<ApprovalSummary>` | yes | Populated after `ApprovalRequested`. |
| | `approvalDecision` | `Optional<String>` | yes | `"APPROVED"` or `"REJECTED"` after `ApprovalDecided`. |
| | `lastTelemetry` | `Optional<TelemetrySnapshot>` | yes | Updated by `TelemetryMonitor` during execution. |
| | `outcome` | `Optional<MissionOutcome>` | yes | Populated after `MissionCompleted`. |
| | `haltReason` | `Optional<HaltReason>` | yes | Populated after `SafetyHaltTriggered`. |
| | `status` | `MissionStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `MissionSubmitted` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Mission` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`ManeuverClass`: `TRIVIAL`, `NON_TRIVIAL`.
`OutcomeStatus`: `COMPLETED`, `PARTIAL`, `ABORTED`.
`MissionStatus`: `SUBMITTED`, `PLANNING`, `PENDING_APPROVAL`, `EXECUTING`, `COMPLETED`, `HALTED`, `FAILED`.

## Events (`MissionEntity`)

| Event | Payload | Transition |
|---|---|---|
| `MissionSubmitted` | `brief` | → SUBMITTED |
| `PlanningStarted` | — | → PLANNING |
| `ApprovalRequested` | `summary` | → PENDING_APPROVAL |
| `ApprovalDecided` | `decision: String` | → EXECUTING (approved) or FAILED (rejected) |
| `MissionStarted` | — | → EXECUTING |
| `WaypointReached` | `waypointId: String` | stays EXECUTING |
| `SafetyHaltTriggered` | `haltReason` | → HALTED (terminal) |
| `MissionCompleted` | `outcome` | → COMPLETED (terminal happy) |
| `MissionFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Mission.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`MissionRow` mirrors `Mission` minus verbose command history. The UI fetches the full mission on demand via `GET /api/missions/{id}` and reads `approvalSummary.proposedCommands` from the JSON for the approval panel.

The view declares ONE query: `getAllMissions: SELECT * AS missions FROM mission_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`MissionTasks.java`)

```java
public final class MissionTasks {
  public static final Task<MissionOutcome> EXECUTE_MISSION = Task
      .name("Execute mission")
      .description("Plan and execute a Spot robot mission from the attached waypoint map; return a MissionOutcome")
      .resultConformsTo(MissionOutcome.class);

  private MissionTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
