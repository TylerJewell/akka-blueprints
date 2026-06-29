# Data model — nexshift

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `Shift` | `shiftId` | `String` | no | Stable id for this open slot. |
| | `role` | `String` | no | Job role label (e.g., `"Nurse"`, `"Technician"`). |
| | `startTime` | `Instant` | no | Shift start (UTC). |
| | `endTime` | `Instant` | no | Shift end (UTC). |
| | `department` | `String` | no | Owning department. |
| | `requiredQualifications` | `List<String>` | no | Qualification codes required; may be empty. |
| `Employee` | `employeeId` | `String` | no | Stable id for this person. |
| | `name` | `String` | no | Display name (first initial + last name). |
| | `qualifications` | `List<String>` | no | Credential codes held. |
| | `availability` | `List<AvailabilityWindow>` | no | Weekly availability windows. |
| | `weeklyHoursCap` | `int` | no | Statutory or contractual maximum hours per week. |
| | `hoursScheduledThisWeek` | `int` | no | Hours already scheduled before this run. |
| `AvailabilityWindow` | `day` | `DayOfWeek` | no | Day of week. |
| | `from` | `LocalTime` | no | Window start. |
| | `to` | `LocalTime` | no | Window end. |
| `ShiftAssignment` | `shiftId` | `String` | no | References a `Shift.shiftId`. |
| | `employeeId` | `String` | no | References an `Employee.employeeId`. |
| | `status` | `AssignmentStatus` | no | Enum value. |
| | `blockedReason` | `Optional<String>` | yes | Non-empty when status is `GUARDRAIL_BLOCKED` or `UNASSIGNED`. |
| `ScheduleRequest` | `scheduleId` | `String` | no | UUID minted by `ScheduleEndpoint`. |
| | `weekStart` | `LocalDate` | no | First day of the scheduling week. |
| | `department` | `String` | no | Department label. |
| | `openShifts` | `List<Shift>` | no | The submitted shift list. |
| | `employees` | `List<Employee>` | no | The submitted roster. |
| | `submittedBy` | `String` | no | Manager identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `ScheduleDraft` | `assignments` | `List<ShiftAssignment>` | no | One entry per open shift. |
| | `totalShifts` | `int` | no | Count of open shifts. |
| | `filledCount` | `int` | no | Assignments with status `ASSIGNED`. |
| | `blockedCount` | `int` | no | Guardrail rejections that could not be recovered. |
| `Schedule` (entity state) | `scheduleId` | `String` | no | — |
| | `request` | `Optional<ScheduleRequest>` | yes | Populated after `ScheduleRequested`. |
| | `draft` | `Optional<ScheduleDraft>` | yes | Populated after `DraftRecorded`. |
| | `status` | `ScheduleStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `ScheduleRequested` emitted. |
| | `confirmedAt` | `Optional<Instant>` | yes | Populated after `ScheduleConfirmed`. |

Every nullable field on `Schedule` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`AssignmentStatus`: `ASSIGNED`, `GUARDRAIL_BLOCKED`, `UNASSIGNED`.
`ScheduleStatus`: `PENDING`, `BUILDING`, `SCHEDULING`, `DRAFT`, `CONFIRMED`, `FAILED`.

## Events (`ScheduleEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ScheduleRequested` | `request` | → PENDING |
| `RosterBuilt` | — | → BUILDING |
| `SchedulingStarted` | — | → SCHEDULING |
| `DraftRecorded` | `draft` | → DRAFT |
| `ScheduleConfirmed` | `confirmedAt: Instant` | → CONFIRMED (terminal happy) |
| `ScheduleFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `Schedule.initial("")` with all `Optional` fields as `Optional.empty()` and `status = PENDING`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`ScheduleRow` mirrors `Schedule`. The UI fetches the full schedule on demand via `GET /api/schedules/{id}` which returns the complete `Schedule` including the nested `ScheduleRequest` and `ScheduleDraft`.

The view declares ONE query: `getAllSchedules: SELECT * AS schedules FROM schedule_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definition (`ScheduleTasks.java`)

```java
public final class ScheduleTasks {
  public static final Task<ScheduleDraft> ASSIGN_SHIFTS = Task
      .name("Assign shifts")
      .description("Fill the open shift list by calling AssignShift for each slot; " +
                   "handle guardrail rejections by trying the next eligible employee")
      .resultConformsTo(ScheduleDraft.class);

  private ScheduleTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).
