# Data model — itsm-change-planner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `ChangeRequest` | `summary` | `String` | no | One-sentence description of the change. |
| | `ciName` | `String` | no | Primary configuration item targeted by the change. |
| | `requestedBy` | `String` | no | Identity of the submitter. |
| | `category` | `ChangeCategory` | no | STANDARD, NORMAL, or EMERGENCY. |
| `HistoricalChange` | `changeId` | `String` | no | Reference ID of the historical record. |
| | `summary` | `String` | no | Summary of the prior change. |
| | `outcome` | `String` | no | succeeded / failed / rolled-back. |
| | `lessonsLearned` | `String` | no | Free-text lessons from the historical record. |
| `ImplementationStep` | `sequence` | `int` | no | 1-based position in the implementation plan. |
| | `description` | `String` | no | One-sentence action. |
| | `targetCi` | `String` | no | CI the step acts on; must be on the allow-list. |
| | `expectedOutcome` | `String` | no | Observable state after the step succeeds. |
| `TestStep` | `sequence` | `int` | no | Matches the corresponding `ImplementationStep`. |
| | `description` | `String` | no | One-sentence test action. |
| | `successCriteria` | `String` | no | Condition that must hold for the test to pass. |
| `BackoutStep` | `sequence` | `int` | no | Reverse-order position (1 undoes the last impl step). |
| | `description` | `String` | no | One-sentence reversal action. |
| | `targetCi` | `String` | no | CI the backout step acts on. |
| `ChangeLedger` | `impactAssessment` | `String` | no | 2–4 sentence assessment of scope and risk. |
| | `similarChanges` | `List<HistoricalChange>` | no | 2–3 relevant historical changes. |
| | `implementationPlan` | `List<ImplementationStep>` | no | Ordered implementation steps (3–6). |
| | `testPlan` | `List<TestStep>` | no | One test step per implementation step. |
| | `backoutPlan` | `List<BackoutStep>` | no | Reverse-order backout steps (3–6). |
| `StepResult` | `sequence` | `int` | no | Step sequence number. |
| | `description` | `String` | no | Echo of the implementation step description. |
| | `ok` | `boolean` | no | True if the executor simulated success. |
| | `evidence` | `String` | no | 3–5 lines of simulated execution evidence. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `TestResult` | `sequence` | `int` | no | Step sequence number. |
| | `passed` | `boolean` | no | True if success criteria were met. |
| | `observation` | `String` | no | 2–3 lines describing what the test observed. |
| | `failureDetail` | `Optional<String>` | yes | Populated when `passed=false`. |
| `BackoutStepResult` | `sequence` | `int` | no | Backout step sequence number. |
| | `ok` | `boolean` | no | True if the backout step succeeded. |
| | `evidence` | `String` | no | 3–5 lines of simulated backout evidence. |
| `StepRecord` | `sequence` | `int` | no | Implementation step sequence number. |
| | `description` | `String` | no | Implementation step description. |
| | `stepResult` | `StepResult` | no | Result from the ExecutorAgent. |
| | `testResult` | `TestResult` | no | Result from the TestRunnerAgent. |
| | `recordedAt` | `Instant` | no | When the record was appended. |
| `ExecutionLog` | `records` | `List<StepRecord>` | no | Append-only log of completed step pairs. |
| `CabDecision` | `outcome` | `CabOutcome` | no | APPROVED or REJECTED. |
| | `reviewedBy` | `String` | no | Identity of the CAB reviewer. |
| | `comments` | `Optional<String>` | yes | Optional reviewer notes. |
| | `decidedAt` | `Instant` | no | When the decision was recorded. |
| `Change` (entity state) | `changeId` | `String` | no | Unique ID. |
| | `summary` | `String` | no | Original change summary. |
| | `ciName` | `String` | no | Primary CI. |
| | `category` | `ChangeCategory` | no | STANDARD / NORMAL / EMERGENCY. |
| | `requestedBy` | `String` | no | Submitter identity. |
| | `status` | `ChangeStatus` | no | See enum. |
| | `ledger` | `Optional<ChangeLedger>` | yes | Populated after `ChangePlanned`. |
| | `executionLog` | `Optional<ExecutionLog>` | yes | Populated after first `StepExecuted`. |
| | `cabDecision` | `Optional<CabDecision>` | yes | Populated after `ChangeApproved` or `ChangeRejected`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `ChangeFailed` / `ChangeFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `ChangeHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `ChangeCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the change reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |

## Enums

- `ChangeCategory` → `STANDARD`, `NORMAL`, `EMERGENCY`.
- `CabOutcome` → `APPROVED`, `REJECTED`.
- `ChangeStatus` → `PLANNING`, `AWAITING_CAB`, `APPROVED`, `EXECUTING`, `ROLLING_BACK`, `IMPLEMENTED`, `REJECTED`, `ROLLED_BACK`, `FAILED`, `HALTED`, `STUCK`.

## Events (`ChangeEntity`)

| Event | Payload | Transition |
|---|---|---|
| `ChangeCreated` | `changeId, summary, ciName, category, requestedBy, createdAt` | → PLANNING |
| `ChangePlanned` | `ledger: ChangeLedger` | → AWAITING_CAB |
| `ChangeApproved` | `decision: CabDecision` | → APPROVED → EXECUTING |
| `ChangeRejected` | `decision: CabDecision` | → REJECTED, `finishedAt = now` |
| `StepBlocked` | `sequence, description, blockerReason` | no status change; notes a blocked step in execution log |
| `StepExecuted` | `sequence, stepResult: StepResult` | no status change; appended to execution log |
| `TestPassed` | `sequence, testResult: TestResult` | no status change; updates `StepRecord` in execution log |
| `TestFailed` | `sequence, testResult: TestResult` | → ROLLING_BACK |
| `BackoutStepExecuted` | `sequence, result: BackoutStepResult` | no status change while ROLLING_BACK |
| `ChangeImplemented` | `finishedAt` | → IMPLEMENTED |
| `ChangeRolledBack` | `finishedAt, incomplete: boolean` | → ROLLED_BACK |
| `ChangeFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `ChangeHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `ChangeFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`ChangeQueue`)

| Event | Payload |
|---|---|
| `ChangeSubmitted` | `changeId, summary, ciName, requestedBy, submittedAt` |

## View row

`ChangeRow` mirrors `Change` but omits `ledger.implementationPlan`, `ledger.testPlan`, and `ledger.backoutPlan` (to keep the list response compact — the UI fetches the full `ChangeLedger` via `GET /api/changes/{id}` on row expand). `executionLog.records` is truncated to the last 3 `StepRecord`s plus a `truncatedFromTotal: int` count; each `StepResult.evidence` is capped at 240 characters. Every nullable field on `ChangeRow` is declared `Optional<T>` (Lesson 6).
