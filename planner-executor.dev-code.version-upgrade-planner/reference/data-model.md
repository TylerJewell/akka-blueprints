# Data model — version-upgrade-planner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `UpgradeRequest` | `sourceVersion` | `String` | no | Airflow version being upgraded from. |
| | `targetVersion` | `String` | no | Airflow version being upgraded to. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `Phase` | `phaseId` | `String` | no | Unique identifier for the phase within the plan. |
| | `name` | `String` | no | Human-readable phase name. |
| | `kind` | `PhaseKind` | no | `COMPATIBILITY_CHECK`, `MIGRATION`, or `TEST_RUN`. |
| | `requiresApproval` | `boolean` | no | True for all `MIGRATION` phases. |
| `PlanLedger` | `sourceVersion` | `String` | no | Source version carried through for executor context. |
| | `targetVersion` | `String` | no | Target version carried through for executor context. |
| | `phases` | `List<Phase>` | no | Ordered list of 2–5 phases. |
| | `currentPhaseIndex` | `int` | no | 0-based index of the phase currently being addressed. |
| | `replanReason` | `Optional<String>` | yes | Set when the planner emits `Replan`; cleared on `Continue`. |
| `PhaseDecision` | `phase` | `Phase` | no | The phase the planner selected. |
| | `executor` | `ExecutorKind` | no | Which executor runs this phase. |
| | `rationale` | `String` | no | One-sentence justification. |
| | `requiresApproval` | `boolean` | no | Mirrors `phase.requiresApproval`. |
| `TestReport` | `total` | `int` | no | Total tests in the suite. |
| | `passed` | `int` | no | Count of passing tests. |
| | `failed` | `int` | no | Count of failing tests. |
| | `failingTests` | `List<String>` | no | Fully-qualified names of failing tests (max 5). |
| `MigrationOutcome` | `phaseId` | `String` | no | Phase that was applied. |
| | `applied` | `boolean` | no | True when all migration steps succeeded. |
| | `summary` | `String` | no | Description of what changed. |
| | `rollbackScript` | `Optional<String>` | yes | Path to the inverse migration script. |
| `PhaseResult` | `executor` | `ExecutorKind` | no | Executor that ran the phase. |
| | `phaseId` | `String` | no | Echo of the phase identifier. |
| | `ok` | `boolean` | no | True when the phase succeeded. |
| | `summary` | `String` | no | Short narrative of what happened. |
| | `testReport` | `Optional<TestReport>` | yes | Populated for `TEST_RUNNER` and `ciGateStep` results. |
| | `migrationOutcome` | `Optional<MigrationOutcome>` | yes | Populated for `MIGRATION_APPLIER` results. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ProgressEntry` | `attempt` | `int` | no | 1-based attempt count for this `(executor, phaseId)` pair. |
| | `executor` | `ExecutorKind` | no | Executor that ran (or would have run) this phase. |
| | `phaseId` | `String` | no | Phase identifier. |
| | `phaseName` | `String` | no | Phase name for display. |
| | `verdict` | `PhaseVerdict` | no | `OK`, `BLOCKED_BY_APPROVAL`, `CI_GATE_FAILED`, or `FAILED`. |
| | `summary` | `String` | no | Narrative from the executor or the gate. |
| | `testReport` | `Optional<TestReport>` | yes | Present on `CI_GATE_FAILED` entries and successful test-run entries. |
| | `blocker` | `Optional<String>` | yes | Populated on `BLOCKED_BY_APPROVAL` and `FAILED`. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ProgressLedger` | `entries` | `List<ProgressEntry>` | no | Append-only. |
| `ApprovalRequest` | `approvalId` | `String` | no | Unique approval identifier. |
| | `jobId` | `String` | no | Parent upgrade job. |
| | `phaseId` | `String` | no | Phase awaiting approval. |
| | `phaseName` | `String` | no | Phase name for reviewer display. |
| | `rationale` | `String` | no | Why this phase requires approval. |
| | `requestedAt` | `Instant` | no | When the approval was requested. |
| `ApprovalDecision` | `approvalId` | `String` | no | The approval being resolved. |
| | `approved` | `boolean` | no | True = approved, false = rejected. |
| | `reviewedBy` | `String` | no | Identity of the reviewer. |
| | `comment` | `Optional<String>` | yes | Required when rejecting; optional when approving. |
| | `decidedAt` | `Instant` | no | When the decision was made. |
| `UpgradeReport` | `summary` | `String` | no | 60–120 word narrative of the completed upgrade. |
| | `appliedPhases` | `List<String>` | no | `phaseId` of each successfully applied migration. |
| | `testsPassed` | `boolean` | no | True when the final `TEST_RUN` phase returned `failed=0`. |
| | `producedAt` | `Instant` | no | When the planner produced the report. |
| `UpgradeJob` (entity state) | `jobId` | `String` | no | Unique job identifier. |
| | `sourceVersion` | `String` | no | Source Airflow version. |
| | `targetVersion` | `String` | no | Target Airflow version. |
| | `requestedBy` | `String` | no | Submitter identifier. |
| | `status` | `JobStatus` | no | See enum. |
| | `ledger` | `Optional<PlanLedger>` | yes | Populated after `JobPlanned`. |
| | `progress` | `Optional<ProgressLedger>` | yes | Populated after first `PhaseRecorded` or `PhaseBlocked`. |
| | `pendingApproval` | `Optional<ApprovalRequest>` | yes | Populated while in `AWAITING_APPROVAL`; cleared on resolution. |
| | `report` | `Optional<UpgradeReport>` | yes | Populated after `JobCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `JobFailed` / `JobFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `JobHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `JobCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `ApprovalState` (entity state) | `request` | `Optional<ApprovalRequest>` | yes | Set when `ApprovalCreated`. |
| | `decision` | `Optional<ApprovalDecision>` | yes | Set when `ApprovalResolved`. |
| `PhaseNextStep` | (sealed interface) | — | — | Permits `Continue(PhaseDecision)`, `Replan(PlanLedger revised)`, `Complete(UpgradeReport stub)`, `Fail(String reason)`. |

## Enums

- `PhaseKind` → `COMPATIBILITY_CHECK`, `MIGRATION`, `TEST_RUN`.
- `ExecutorKind` → `COMPAT_CHECKER`, `TEST_RUNNER`, `MIGRATION_APPLIER`.
- `PhaseVerdict` → `OK`, `BLOCKED_BY_APPROVAL`, `CI_GATE_FAILED`, `FAILED`.
- `JobStatus` → `PLANNING`, `EXECUTING`, `AWAITING_APPROVAL`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`UpgradeJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, sourceVersion, targetVersion, requestedBy, createdAt` | → PLANNING |
| `JobPlanned` | `ledger: PlanLedger` | → EXECUTING |
| `PhaseDispatched` | `phaseId, executor` | no status change |
| `ApprovalRequested` | `approvalId, phaseId, phaseName, rationale, requestedAt` | → AWAITING_APPROVAL; sets `pendingApproval` |
| `ApprovalGranted` | `approvalId, reviewedBy, decidedAt` | → EXECUTING; clears `pendingApproval` |
| `ApprovalRejected` | `approvalId, reviewedBy, comment, decidedAt` | → EXECUTING; clears `pendingApproval`; appends PhaseBlocked entry |
| `PhaseBlocked` | `attempt, phaseId, phaseName, blocker` | no status change; appends `ProgressEntry{verdict=BLOCKED_BY_APPROVAL}` |
| `PhaseRecorded` | `entry: ProgressEntry` | no status change; appends to `progress.entries` |
| `CiGateFailed` | `phaseId, testReport` | no status change; appends `ProgressEntry{verdict=CI_GATE_FAILED}` |
| `LedgerRevised` | `ledger: PlanLedger` | no status change; replaces `ledger` |
| `JobCompleted` | `report: UpgradeReport` | → COMPLETED, `finishedAt = now` |
| `JobFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `JobHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`ApprovalEntity`)

| Event | Payload |
|---|---|
| `ApprovalCreated` | `approvalId, jobId, phaseId, phaseName, rationale, requestedAt` |
| `ApprovalResolved` | `approvalId, approved, reviewedBy, comment, decidedAt` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`UpgradeRequestQueue`)

| Event | Payload |
|---|---|
| `UpgradeSubmitted` | `jobId, sourceVersion, targetVersion, requestedBy, submittedAt` |

## View row

`UpgradeJobRow` mirrors `UpgradeJob` minus the heavy progress payload — `progress.entries` is truncated to the last 3 entries plus `truncatedFromTotal: int`, and each `summary` is capped at 200 characters. The UI fetches the full job by id on click via `GET /api/jobs/{id}`. Every nullable field is declared `Optional<T>` (Lesson 6).
