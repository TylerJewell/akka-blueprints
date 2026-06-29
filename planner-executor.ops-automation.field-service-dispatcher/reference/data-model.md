# Data model — field-service-dispatcher

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `WorkOrderRequest` | `description` | `String` | no | Work order description submitted by the user. |
| | `serviceAddress` | `String` | no | Physical address for the job. |
| | `requiredSkill` | `String` | no | Skill required to service the work order (e.g., `plumbing`, `hvac`). |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ScheduleLedger` | `knownOrders` | `List<String>` | no | Work orders the dispatcher believes are in scope for this schedule. |
| | `unassignedSlots` | `List<String>` | no | Work order ids still lacking a confirmed technician assignment. |
| | `plan` | `List<String>` | no | Ordered list of assignment steps (2–6). |
| | `currentDispatch` | `Optional<AssignmentDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `AssignmentDecision` | `specialist` | `SpecialistKind` | no | Which specialist handles the assignment step. |
| | `assignment` | `String` | no | One-sentence description of the proposed assignment. |
| | `rationale` | `String` | no | One-sentence justification. |
| `AssignmentResult` | `specialist` | `SpecialistKind` | no | Specialist that ran the step. |
| | `assignment` | `String` | no | Echo of the assignment. |
| | `ok` | `boolean` | no | True if the specialist could fulfill the step. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `AssignmentEntry` | `attempt` | `int` | no | 1-based attempt count for this `(specialist, assignment)` pair. |
| | `specialist` | `SpecialistKind` | no | Specialist that ran (or would have run) this step. |
| | `assignment` | `String` | no | The assignment text. |
| | `verdict` | `AssignmentVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / FAIRNESS_FLAGGED. |
| | `scrubbedResult` | `String` | no | Sanitized result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `FairnessAlert` | `technicianId` | `String` | no | Technician id that triggered the alert. |
| | `alertType` | `String` | no | `"overloaded"` or `"underassigned"`. |
| | `detail` | `String` | no | Human-readable explanation of the imbalance. |
| | `detectedAt` | `Instant` | no | When the monitor detected the imbalance. |
| `FairnessLedger` | `entries` | `List<AssignmentEntry>` | no | Append-only assignment record. |
| | `alerts` | `List<FairnessAlert>` | no | Advisory alerts recorded by `FairnessMonitor`. |
| `DispatchSummary` | `overview` | `String` | no | 60–120 word summary of the dispatch outcome. |
| | `assignments` | `List<String>` | no | 2–4 cited bullets, each tagged by specialist. |
| | `producedAt` | `Instant` | no | When the dispatcher produced the summary. |
| `Schedule` (entity state) | `scheduleId` | `String` | no | Unique id. |
| | `description` | `String` | no | Original work order description. |
| | `serviceAddress` | `String` | no | Service address. |
| | `requiredSkill` | `String` | no | Required skill. |
| | `status` | `ScheduleStatus` | no | See enum. |
| | `ledger` | `Optional<ScheduleLedger>` | yes | Populated after `SchedulePlanned`. |
| | `fairness` | `Optional<FairnessLedger>` | yes | Populated after first `AssignmentRecorded` or `AssignmentBlocked`. |
| | `summary` | `Optional<DispatchSummary>` | yes | Populated after `ScheduleCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `ScheduleFailed` / `ScheduleFailedTimeout`. |
| | `pauseReason` | `Optional<String>` | yes | Populated on `SchedulePausedOperator`. |
| | `createdAt` | `Instant` | no | When `ScheduleCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the schedule reached a terminal state. |
| `OperatorControl` (entity state) | `paused` | `boolean` | no | Operator pause flag. |
| | `reason` | `Optional<String>` | yes | Set when `PauseRequested`. |
| | `pausedAt` | `Optional<Instant>` | yes | Set when `PauseRequested`; cleared when `PauseCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(AssignmentDecision)`, `Replan(ScheduleLedger revised)`, `Complete(DispatchSummary stub)`, `Fail(String reason)`. |

## Enums

- `SpecialistKind` → `ROUTE_OPTIMIZER`, `AVAILABILITY`.
- `AssignmentVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `FAIRNESS_FLAGGED`.
- `ScheduleStatus` → `PLANNING`, `DISPATCHING`, `COMPLETED`, `FAILED`, `PAUSED`, `STALLED`.

## Events (`ScheduleEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ScheduleCreated` | `scheduleId, description, serviceAddress, requiredSkill, createdAt` | → PLANNING |
| `SchedulePlanned` | `ledger` | → DISPATCHING |
| `AssignmentDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `AssignmentBlocked` | `attempt, dispatch, blocker` | no status change; appends an `AssignmentEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `AssignmentRecorded` | `entry: AssignmentEntry` | no status change; appends to `fairness.entries`. |
| `LedgerRevised` | `ledger: ScheduleLedger` | no status change; replaces `ledger`. |
| `FairnessAlertRecorded` | `alert: FairnessAlert` | no status change; appends to `fairness.alerts`. |
| `ScheduleCompleted` | `summary` | → COMPLETED, `finishedAt = now` |
| `ScheduleFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `SchedulePausedOperator` | `pauseReason` | → PAUSED, `finishedAt = now` |
| `ScheduleFailedTimeout` | `failureReason` | → STALLED, `finishedAt = now` |

## Events (`OperatorControlEntity`)

| Event | Payload |
|---|---|
| `PauseRequested` | `reason, pausedAt` |
| `PauseCleared` | `clearedAt` |

## Events (`WorkOrderQueue`)

| Event | Payload |
|---|---|
| `WorkOrderSubmitted` | `scheduleId, description, serviceAddress, requiredSkill, requestedBy, submittedAt` |

## View row

`ScheduleRow` mirrors `Schedule` minus the heavy fairness payload — `fairness.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedResult` is capped at 240 characters. `fairness.alerts` is included in full (alerts are short records). The UI fetches the full schedule by id on click via `GET /api/schedules/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
