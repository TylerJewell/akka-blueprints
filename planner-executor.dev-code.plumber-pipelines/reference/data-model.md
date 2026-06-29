# Data model — plumber-data-engineering-assistant

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `PipelineRequest` | `prompt` | `String` | no | User-submitted pipeline request. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `PipelineLedger` | `sources` | `List<String>` | no | Declared input datasets with format and schema id. |
| | `transforms` | `List<String>` | no | Ordered list of transformation steps. |
| | `sinks` | `List<String>` | no | Declared output datasets with format and schema id. |
| | `validationPlan` | `List<String>` | no | Schema checks the SchemaAgent must run before finalisation. |
| | `currentStage` | `Optional<StageDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `StageDecision` | `engine` | `EngineKind` | no | Which specialist runs the stage. |
| | `stageKind` | `StageKind` | no | Phase of the stage: SOURCE, TRANSFORM, SINK, or VALIDATE. |
| | `spec` | `String` | no | Description of the stage in enough detail for the specialist. |
| | `rationale` | `String` | no | One-sentence justification. |
| `StageResult` | `engine` | `EngineKind` | no | Specialist that ran the stage. |
| | `stageKind` | `StageKind` | no | Phase of the stage. |
| | `ok` | `boolean` | no | True if the specialist could produce a valid artifact. |
| | `artifact` | `String` | no | Raw JSON or YAML artifact (pre-test-gate). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ProgressEntry` | `attempt` | `int` | no | 1-based attempt count for this `(engine, stageKind)` pair. |
| | `engine` | `EngineKind` | no | Specialist that ran (or was blocked for) this stage. |
| | `stageKind` | `StageKind` | no | Phase of the stage. |
| | `verdict` | `ProgressVerdict` | no | OK / BLOCKED_BY_TEST_GATE / FAILED / SCHEMA_ERROR. |
| | `artifact` | `String` | no | Scrubbed artifact text (after `SecretScrubber`). |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ProgressLedger` | `entries` | `List<ProgressEntry>` | no | Append-only. |
| `PipelineDefinition` | `title` | `String` | no | Short pipeline name. |
| | `description` | `String` | no | 60–120 word description of the end-to-end flow. |
| | `stages` | `List<String>` | no | Ordered stages from the progress ledger. |
| | `sinkSchema` | `String` | no | Declared sink schema id from the pipeline ledger. |
| | `producedAt` | `Instant` | no | When the planner produced the definition. |
| `Pipeline` (entity state) | `pipelineId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original prompt. |
| | `status` | `PipelineStatus` | no | See enum. |
| | `ledger` | `Optional<PipelineLedger>` | yes | Populated after `PipelinePlanned`. |
| | `progress` | `Optional<ProgressLedger>` | yes | Populated after first `StageRecorded` or `StageBlocked`. |
| | `definition` | `Optional<PipelineDefinition>` | yes | Populated after `PipelineFinalised`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `PipelineFailed` / `PipelineFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `PipelineHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `PipelineCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the pipeline reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(StageDecision)`, `Replan(PipelineLedger revised)`, `Finalise(PipelineDefinition stub)`, `Fail(String reason)`. |

## Enums

- `EngineKind` → `SPARK`, `BEAM`, `DBT`, `SCHEMA`.
- `StageKind` → `SOURCE`, `TRANSFORM`, `SINK`, `VALIDATE`.
- `ProgressVerdict` → `OK`, `BLOCKED_BY_TEST_GATE`, `FAILED`, `SCHEMA_ERROR`.
- `PipelineStatus` → `PLANNING`, `EXECUTING`, `FINALISED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`PipelineEntity`)

| Event | Payload | Transition |
|---|---|---|
| `PipelineCreated` | `pipelineId, prompt, createdAt` | → PLANNING |
| `PipelinePlanned` | `ledger` | → EXECUTING |
| `StageDispatched` | `decision` | no status change; sets `ledger.currentStage`. |
| `StageBlocked` | `attempt, decision, blocker` | no status change; appends a `ProgressEntry` with verdict `BLOCKED_BY_TEST_GATE`. |
| `StageRecorded` | `entry: ProgressEntry` | no status change; appends to `progress.entries`. |
| `LedgerRevised` | `ledger: PipelineLedger` | no status change; replaces `ledger`. |
| `PipelineFinalised` | `definition` | → FINALISED, `finishedAt = now` |
| `PipelineFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `PipelineHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `PipelineFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `PipelineSubmitted` | `pipelineId, prompt, requestedBy, submittedAt` |

## View row

`PipelineRow` mirrors `Pipeline` minus the heavy progress payload — `progress.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `artifact` is capped at 240 characters. The UI fetches the full pipeline by id on click via `GET /api/pipelines/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
