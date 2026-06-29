# Data model — sdlc-task-planner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PlanRequest` | `prompt` | `String` | no | User-submitted SDLC request prompt. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `TaskLedger` | `requirements` | `List<String>` | no | Requirements the planner has identified. |
| | `constraints` | `List<String>` | no | Constraints (scope limits, tech boundaries) the planner has identified. |
| | `plan` | `List<String>` | no | Ordered list of plan steps (3–8). |
| | `currentDispatch` | `Optional<DispatchDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `DispatchDecision` | `specialist` | `SpecialistKind` | no | Which specialist runs the sub-task. |
| | `subtask` | `String` | no | One-sentence sub-task description. |
| | `rationale` | `String` | no | One-sentence justification. |
| `SubtaskResult` | `specialist` | `SpecialistKind` | no | Specialist that ran the sub-task. |
| | `subtask` | `String` | no | Echo of the sub-task. |
| | `ok` | `boolean` | no | True if the specialist could fulfill the sub-task. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ProgressEntry` | `attempt` | `int` | no | 1-based attempt count for this `(specialist, subtask)` pair. |
| | `specialist` | `SpecialistKind` | no | Specialist that ran (or would have run) this sub-task. |
| | `subtask` | `String` | no | The sub-task text. |
| | `verdict` | `ProgressVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `scrubbedResult` | `String` | no | Sanitized result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ProgressLedger` | `entries` | `List<ProgressEntry>` | no | Append-only. |
| `PlanDeliverable` | `summary` | `String` | no | 60–120 word summary of the plan outcomes. |
| | `artefacts` | `List<String>` | no | 3–5 cited artefact bullets, each tagged by specialist. |
| | `producedAt` | `Instant` | no | When the planner produced the deliverable. |
| `Plan` (entity state) | `planId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original request prompt. |
| | `status` | `PlanStatus` | no | See enum. |
| | `ledger` | `Optional<TaskLedger>` | yes | Populated after `PlanDecomposed`. |
| | `progress` | `Optional<ProgressLedger>` | yes | Populated after first `SubtaskRecorded` or `SubtaskBlocked`. |
| | `deliverable` | `Optional<PlanDeliverable>` | yes | Populated after `PlanCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `PlanFailed` / `PlanFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `PlanHaltedOperator` / `PlanHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `PlanCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the plan reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(DispatchDecision)`, `Replan(TaskLedger revised)`, `Complete(PlanDeliverable stub)`, `Fail(String reason)`. |

## Enums

- `SpecialistKind` → `ANALYST`, `ARCHITECT`, `CODER`, `REVIEWER`.
- `ProgressVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `PlanStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`PlanEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PlanCreated` | `planId, prompt, createdAt` | → PLANNING |
| `PlanDecomposed` | `ledger` | → EXECUTING |
| `SubtaskDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `SubtaskBlocked` | `attempt, dispatch, blocker` | no status change; appends a `ProgressEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `SubtaskRecorded` | `entry: ProgressEntry` | no status change; appends to `progress.entries`. |
| `LedgerRevised` | `ledger: TaskLedger` | no status change; replaces `ledger`. |
| `PlanCompleted` | `deliverable` | → COMPLETED, `finishedAt = now` |
| `PlanFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `PlanHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `PlanHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `PlanFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `PlanSubmitted` | `planId, prompt, requestedBy, submittedAt` |

## View row

`PlanRow` mirrors `Plan` minus the heavy progress payload — `progress.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full plan by id on click via `GET /api/plans/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
