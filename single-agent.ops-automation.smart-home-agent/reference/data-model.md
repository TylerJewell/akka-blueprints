# Data model — smart-home-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Device` | `deviceId` | `String` | no | Stable identifier (e.g. `kitchen-light`). |
| | `displayName` | `String` | no | Human-readable label (e.g. `Kitchen Light`). |
| | `deviceClass` | `DeviceClass` | no | Enum value. |
| | `location` | `String` | no | Room or area (e.g. `kitchen`, `front door`). |
| | `minValue` | `int` | no | Lower bound for numeric params; 0 for non-numeric devices. |
| | `maxValue` | `int` | no | Upper bound; 0 for non-numeric devices. |
| `DeviceState` | `deviceId` | `String` | no | Matches a `Device.deviceId`. |
| | `on` | `boolean` | no | Applies to LIGHT and BLIND. |
| | `brightnessPercent` | `Integer` | yes | LIGHT only; null for other classes. |
| | `temperatureSetPoint` | `Integer` | yes | THERMOSTAT only; null for other classes. |
| | `locked` | `boolean` | no | LOCK only; false for other classes. |
| `CommandRequest` | `commandId` | `String` | no | UUID minted by `CommandEndpoint`. |
| | `homeId` | `String` | no | Home identifier (default `home-1`). |
| | `rawCommand` | `String` | no | Resident's natural-language input. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `DeviceAction` | `deviceId` | `String` | no | MUST match a `Device.deviceId`, or `"unknown"`. |
| | `actionType` | `ActionType` | no | Enum value. |
| | `numericParam` | `Integer` | yes | Brightness %, temperature °F, or null. |
| | `rationale` | `String` | no | One sentence from the agent. |
| `GuardResult` | `passed` | `boolean` | no | Whether the guardrail accepted the action. |
| | `rejectionReason` | `String` | yes | Named check that failed; null when passed. |
| `Command` (entity state) | `commandId` | `String` | no | — |
| | `request` | `Optional<CommandRequest>` | yes | Populated after `CommandReceived`. |
| | `action` | `Optional<DeviceAction>` | yes | Populated after `GuardPassed`. |
| | `guardResult` | `Optional<GuardResult>` | yes | Populated after `GuardPassed` or `CommandBlocked`. |
| | `status` | `CommandStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `CommandReceived` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `Command` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`DeviceClass`: `LIGHT`, `THERMOSTAT`, `LOCK`, `BLIND`.

`ActionType`: `TURN_ON`, `TURN_OFF`, `SET_BRIGHTNESS`, `SET_TEMPERATURE`, `LOCK`, `UNLOCK`, `OPEN`, `CLOSE`.

`CommandStatus`: `RECEIVED`, `GUARD_PASSED`, `BLOCKED`, `AWAITING_CONFIRMATION`, `DISPATCHED`, `CANCELLED`, `FAILED`.

## Events (`CommandEntity`)

| Event | Payload | Transition |
|---|---|---|
| `CommandReceived` | `request` | → RECEIVED |
| `GuardPassed` | `action`, `guardResult` | → GUARD_PASSED |
| `CommandBlocked` | `reason: String`, `guardResult` | → BLOCKED (terminal) |
| `ConfirmationRequested` | `action` | → AWAITING_CONFIRMATION |
| `ActionConfirmed` | — | stays AWAITING_CONFIRMATION (workflow advances) |
| `ActionCancelled` | — | → CANCELLED (terminal) |
| `ActionDispatched` | `action` | → DISPATCHED (terminal happy) |
| `CommandFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Command.initial("")` with all `Optional` fields as `Optional.empty()` and `status = RECEIVED`. `emptyState()` never references `commandContext()` (Lesson 3).

## Events (`DeviceRegistry`)

| Event | Payload | Effect |
|---|---|---|
| `DeviceRegistered` | `device: Device` | Adds device to catalogue map. |
| `DeviceStateChanged` | `deviceId: String`, `newState: DeviceState` | Updates current state map. |

## View row

`CommandRow` mirrors `Command`. One row per command in `command_view`.

The view declares ONE query: `getAllCommands: SELECT * AS commands FROM command_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`CommandTasks.java`)

```java
public final class CommandTasks {
  public static final Task<DeviceAction> CONTROL_DEVICE = Task
      .name("Control device")
      .description("Parse the resident's command, select the target device from the catalogue, and return a DeviceAction")
      .resultConformsTo(DeviceAction.class);

  private CommandTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
