# Data model — bq-pipeline-builder

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PipelineRequest` | `description` | `String` | no | User-submitted natural-language pipeline description. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `PipelineLedger` | `datasetFacts` | `List<String>` | no | Facts the planner believes are established about the source and target datasets. |
| | `schemaGaps` | `List<String>` | no | Schema questions the planner believes are still unresolved. |
| | `buildPlan` | `List<String>` | no | Ordered list of build steps (3–8). |
| | `currentDispatch` | `Optional<BuildStepDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `BuildStepDecision` | `executor` | `ExecutorKind` | no | Which specialist executor runs the step. |
| | `step` | `String` | no | One-sentence step description. |
| | `rationale` | `String` | no | One-sentence justification. |
| `StepOutput` | `executor` | `ExecutorKind` | no | Executor that ran the step. |
| | `step` | `String` | no | Echo of the step. |
| | `ok` | `boolean` | no | True if the executor could fulfil the step. |
| | `content` | `String` | no | Raw textual output (SQL, SQLX, JS macro, or validation report JSON). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `StepEntry` | `attempt` | `int` | no | 1-based attempt count for this `(executor, step)` pair. |
| | `executor` | `ExecutorKind` | no | Executor that ran (or would have run) this step. |
| | `step` | `String` | no | The step text. |
| | `verdict` | `StepVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / CI_GATE_FAILED. |
| | `output` | `String` | no | Executor output or gate failure description. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `StepLedger` | `entries` | `List<StepEntry>` | no | Append-only. |
| `ValidationReport` | `passed` | `boolean` | no | True when failures is empty. |
| | `failures` | `List<String>` | no | One sentence per blocking defect. |
| | `warnings` | `List<String>` | no | One sentence per non-blocking concern. |
| `BuildManifest` | `summary` | `String` | no | 60–120 word summary of the pipeline built. |
| | `artifacts` | `List<String>` | no | 3–5 artifact bullets, each tagged with executor kind. |
| | `producedAt` | `Instant` | no | When the planner produced the manifest. |
| `Pipeline` (entity state) | `pipelineId` | `String` | no | Unique id. |
| | `description` | `String` | no | Original description. |
| | `status` | `PipelineStatus` | no | See enum. |
| | `ledger` | `Optional<PipelineLedger>` | yes | Populated after `PipelinePlanned`. |
| | `steps` | `Optional<StepLedger>` | yes | Populated after first `StepRecorded` or `StepBlocked`. |
| | `manifest` | `Optional<BuildManifest>` | yes | Populated after `PipelineCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `PipelineFailed` / `PipelineFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `PipelineHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `PipelineCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the pipeline reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextBuildStep` | (sealed interface) | — | — | Permits `Continue(BuildStepDecision)`, `Replan(PipelineLedger revised)`, `Complete(BuildManifest stub)`, `Fail(String reason)`. |

## Enums

- `ExecutorKind` → `SCHEMA_ANALYST`, `SQL_COMPOSER`, `DATAFORM_MODELER`, `VALIDATOR`.
- `StepVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `CI_GATE_FAILED`.
- `PipelineStatus` → `PLANNING`, `BUILDING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`PipelineEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PipelineCreated` | `pipelineId, description, createdAt` | → PLANNING |
| `PipelinePlanned` | `ledger` | → BUILDING |
| `StepDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `StepBlocked` | `attempt, dispatch, blocker` | no status change; appends a `StepEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `StepRecorded` | `entry: StepEntry` | no status change; appends to `steps.entries`. |
| `LedgerRevised` | `ledger: PipelineLedger` | no status change; replaces `ledger`. |
| `PipelineCompleted` | `manifest` | → COMPLETED, `finishedAt = now` |
| `PipelineFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `PipelineHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `PipelineFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`PipelineQueue`)

| Event | Payload |
|---|---|
| `PipelineSubmitted` | `pipelineId, description, requestedBy, submittedAt` |

## View row

`PipelineRow` mirrors `Pipeline` minus the heavy step payload — `steps.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `output` is capped at 240 characters. The UI fetches the full pipeline by id on click via `GET /api/pipelines/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
