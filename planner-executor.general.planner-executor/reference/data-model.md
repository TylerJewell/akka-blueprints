# Data model — planner-executor-general-planner-executor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GoalRequest` | `goal` | `String` | no | User-submitted goal text. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `PlanLedger` | `goal` | `String` | no | The original goal text, carried forward for context on every decide call. |
| | `openSteps` | `List<String>` | no | Steps not yet executed or blocked. |
| | `completedSteps` | `List<String>` | no | Steps whose `StepEntry` carries `verdict = OK`. |
| | `blockedSteps` | `List<String>` | no | Steps whose `StepEntry` carries `verdict = BLOCKED_BY_GUARDRAIL`. |
| | `activeStep` | `Optional<String>` | yes | Populated between `proposeStep` and `recordStep`; null otherwise. |
| `StepDecision` | `step` | `String` | no | The exact step text the Executor will receive. |
| | `rationale` | `String` | no | One-sentence justification from the Planner. |
| `StepResult` | `step` | `String` | no | Echo of the step text. |
| | `ok` | `boolean` | no | True if the Executor could fulfill the step. |
| | `content` | `String` | no | Textual result from the Executor. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `StepEntry` | `attempt` | `int` | no | 1-based attempt count for this step text. |
| | `step` | `String` | no | The step text. |
| | `verdict` | `StepVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `result` | `String` | no | Executor result text (or block reason if guardrail fired). |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED_BY_GUARDRAIL or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `StepLog` | `entries` | `List<StepEntry>` | no | Append-only log of all step attempts. |
| `PlanOutcome` | `summary` | `String` | no | 60–100 word summary produced by the Planner on completion. |
| | `completedSteps` | `List<String>` | no | Steps in the order they were completed, from the step log. |
| | `producedAt` | `Instant` | no | When the Planner produced the outcome. |
| `Plan` (entity state) | `planId` | `String` | no | Unique id. |
| | `goal` | `String` | no | Original goal text. |
| | `status` | `PlanStatus` | no | See enum. |
| | `ledger` | `Optional<PlanLedger>` | yes | Populated after `PlanStarted`. |
| | `stepLog` | `Optional<StepLog>` | yes | Populated after first `StepRecorded` or `StepBlocked`. |
| | `outcome` | `Optional<PlanOutcome>` | yes | Populated after `PlanCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `PlanFailed` / `PlanFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `PlanHaltedOperator` / `PlanHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `PlanCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the plan reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextAction` | (sealed interface) | — | — | Permits `Continue(StepDecision)`, `Replan(PlanLedger revised)`, `Complete(PlanOutcome stub)`, `Fail(String reason)`. |

## Enums

- `StepVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `PlanStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`PlanEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PlanCreated` | `planId, goal, createdAt` | → PLANNING |
| `PlanStarted` | `ledger` | → EXECUTING |
| `StepProposed` | `decision: StepDecision` | no status change; sets `ledger.activeStep`. |
| `StepBlocked` | `attempt, step, blocker` | no status change; appends a `StepEntry` with `verdict = BLOCKED_BY_GUARDRAIL`. |
| `StepRecorded` | `entry: StepEntry` | no status change; appends to `stepLog.entries`; updates `ledger.completedSteps` or `ledger.blockedSteps`. |
| `PlanRevised` | `ledger: PlanLedger` | no status change; replaces `ledger`. |
| `PlanCompleted` | `outcome: PlanOutcome` | → COMPLETED, `finishedAt = now` |
| `PlanFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `PlanHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `PlanHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `PlanFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`GoalQueue`)

| Event | Payload |
|---|---|
| `GoalSubmitted` | `planId, goal, requestedBy, submittedAt` |

## View row

`PlanRow` mirrors `Plan` minus the heavy step log payload — `stepLog.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `result` is capped at 240 characters. The UI fetches the full plan by id on click via `GET /api/plans/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
