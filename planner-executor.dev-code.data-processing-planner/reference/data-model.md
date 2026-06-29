# Data model — data-processing-planner

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PipelineRequest` | `description` | `String` | no | User-submitted pipeline description. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| | `targetDataset` | `String` | no | Name or S3 prefix of the dataset to process. |
| `JobLedger` | `knownInputs` | `List<String>` | no | Data sources and parameters the planner believes are known. |
| | `missingParams` | `List<String>` | no | Parameters the planner still needs to discover. |
| | `jobPlan` | `List<String>` | no | Ordered list of job plan steps (3–6). |
| | `currentDispatch` | `Optional<JobDispatch>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `JobDispatch` | `engine` | `JobEngine` | no | Which engine runs the step. |
| | `stepSpec` | `String` | no | One-sentence step description. |
| | `estimatedCostUsd` | `double` | no | Planner's estimated cost for this step in USD. |
| | `rationale` | `String` | no | One-sentence justification for the choice. |
| `JobStepResult` | `engine` | `JobEngine` | no | Engine that ran the step. |
| | `stepSpec` | `String` | no | Echo of the step spec. |
| | `ok` | `boolean` | no | True if the step could fulfill the specification. |
| | `output` | `String` | no | Raw textual output (pre-sanitize). |
| | `actualCostUsd` | `double` | no | Actual cost reported by the fixture or simulated engine. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `RunEntry` | `attempt` | `int` | no | 1-based attempt count for this `(engine, stepSpec)` pair. |
| | `engine` | `JobEngine` | no | Engine that ran (or would have run) this step. |
| | `stepSpec` | `String` | no | The step specification text. |
| | `verdict` | `RunVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / COST_EXCEEDED. |
| | `scrubbedOutput` | `String` | no | Sanitized output text. |
| | `costConsumed` | `double` | no | Actual cost charged; 0.0 on BLOCKED_BY_GUARDRAIL. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `RunLedger` | `entries` | `List<RunEntry>` | no | Append-only. |
| `PipelineOutput` | `outputLocation` | `String` | no | S3 path (or best-effort synthetic path) of the final output. |
| | `rowsProcessed` | `long` | no | Total rows processed across all steps; -1 if not determinable. |
| | `totalCostUsd` | `double` | no | Sum of `costConsumed` across all run entries. |
| | `summary` | `String` | no | 60–120 word narrative summary. |
| | `producedAt` | `Instant` | no | When the planner produced the output. |
| `PipelineRun` (entity state) | `runId` | `String` | no | Unique id. |
| | `description` | `String` | no | Original pipeline description. |
| | `targetDataset` | `String` | no | Original target dataset. |
| | `status` | `PipelineStatus` | no | See enum. |
| | `ledger` | `Optional<JobLedger>` | yes | Populated after `PipelineRunPlanned`. |
| | `runLedger` | `Optional<RunLedger>` | yes | Populated after first `JobStepRecorded` or `JobStepBlocked`. |
| | `output` | `Optional<PipelineOutput>` | yes | Populated after `PipelineRunCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `PipelineRunFailed` / `PipelineRunFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `PipelineRunHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `PipelineRunCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `ContinueDispatch(JobDispatch)`, `RevisePlan(JobLedger revised)`, `Complete(PipelineOutput stub)`, `Fail(String reason)`. |

## Enums

- `JobEngine` → `GLUE_CRAWLER`, `EMR_STEP`, `SPARK_SUBMIT`, `S3_COPY`.
- `RunVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `COST_EXCEEDED`.
- `PipelineStatus` → `PLANNING`, `RUNNING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`PipelineEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PipelineRunCreated` | `runId, description, targetDataset, createdAt` | → PLANNING |
| `PipelineRunPlanned` | `ledger: JobLedger` | → RUNNING |
| `JobStepDispatched` | `dispatch: JobDispatch` | no status change; sets `ledger.currentDispatch`. |
| `JobStepBlocked` | `attempt, dispatch, blocker` | no status change; appends a `RunEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `JobStepRecorded` | `entry: RunEntry` | no status change; appends to `runLedger.entries`. |
| `JobLedgerRevised` | `ledger: JobLedger` | no status change; replaces `ledger`. |
| `PipelineRunCompleted` | `output: PipelineOutput` | → COMPLETED, `finishedAt = now` |
| `PipelineRunFailed` | `failureReason: String` | → FAILED, `finishedAt = now` |
| `PipelineRunHaltedOperator` | `haltReason: String` | → HALTED, `finishedAt = now` |
| `PipelineRunFailedTimeout` | `failureReason: String` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`PipelineQueue`)

| Event | Payload |
|---|---|
| `PipelineSubmitted` | `runId, description, targetDataset, requestedBy, submittedAt` |

## View row

`PipelineRunRow` mirrors `PipelineRun` minus the heavy run payload — `runLedger.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedOutput` is capped at 240 characters. The UI fetches the full run by id on click via `GET /api/pipelines/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
