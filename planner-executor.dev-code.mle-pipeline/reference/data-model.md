# Data model — mle-pipeline

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `RunRequest` | `datasetRef` | `String` | no | Identifier for the dataset fixture to use. |
| | `objective` | `String` | no | ML objective: classification, regression, or anomaly detection. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `PipelineLedger` | `facts` | `List<String>` | no | Facts the planner believes are known about the dataset and objective. |
| | `gaps` | `List<String>` | no | Facts the planner still needs to close before promotion. |
| | `stages` | `List<String>` | no | Ordered list of pipeline stages (4–6). |
| | `activeDispatch` | `Optional<StageDispatch>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `StageDispatch` | `specialist` | `SpecialistKind` | no | Which specialist runs the stage. |
| | `stageName` | `String` | no | Short identifier for the stage (e.g., `train-classifier`). |
| | `instruction` | `String` | no | One-sentence task instruction for the specialist. |
| | `rationale` | `String` | no | One-sentence justification for this dispatch. |
| `MetricBundle` | `accuracy` | `double` | no | Proportion of correct predictions on the held-out eval set. |
| | `f1` | `double` | no | Harmonic mean of precision and recall. |
| | `auc` | `double` | no | Area under the ROC curve. |
| | `fairnessDelta` | `double` | no | Maximum group disparity on the protected attribute across all groups. |
| | `notes` | `Optional<String>` | yes | Any caveats or evaluation conditions. |
| `StageResult` | `specialist` | `SpecialistKind` | no | Specialist that ran the stage. |
| | `stageName` | `String` | no | Echo of the stage name. |
| | `ok` | `boolean` | no | True if the specialist could fulfill the stage. |
| | `content` | `String` | no | Textual result (profile stats, feature plan JSON, training metadata, or evaluation summary). |
| | `metrics` | `Optional<MetricBundle>` | yes | Populated only by `MODEL_EVALUATOR` when `ok = true`. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok = false`. |
| `EvalEntry` | `attempt` | `int` | no | 1-based attempt count for this `(specialist, stageName)` pair. |
| | `specialist` | `SpecialistKind` | no | Specialist that ran (or attempted) this stage. |
| | `stageName` | `String` | no | The stage identifier. |
| | `verdict` | `EvalVerdict` | no | OK / GATE_FAILED / FAILED / BLOCKED. |
| | `metrics` | `Optional<MetricBundle>` | yes | Populated when specialist is MODEL_EVALUATOR and ok = true. |
| | `gateOutcome` | `GateOutcome` | no | PASSED / FAILED / NOT_APPLICABLE. |
| | `blocker` | `Optional<String>` | yes | Populated on GATE_FAILED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `EvaluationLedger` | `entries` | `List<EvalEntry>` | no | Append-only. |
| `ModelReport` | `summary` | `String` | no | 60–120 word summary citing dataset, feature strategy, training config, and eval metrics. |
| | `finalMetrics` | `MetricBundle` | no | The MetricBundle from the passing MODEL_EVALUATOR entry. |
| | `stages` | `List<String>` | no | Ordered list of stage names from the evaluation ledger. |
| | `producedAt` | `Instant` | no | When the planner produced the report. |
| `DriftAlert` | `runId` | `String` | no | The run whose metrics triggered the alert. |
| | `metricName` | `String` | no | Name of the drifted metric (accuracy, f1, auc, fairnessDelta). |
| | `baseline` | `double` | no | Baseline metric value from `metrics-baseline.json`. |
| | `observed` | `double` | no | Observed metric value from the completed run. |
| | `delta` | `double` | no | Absolute difference `|observed - baseline|`. |
| | `detectedAt` | `Instant` | no | When DriftWatchMonitor detected the drift. |
| `PipelineRun` (entity state) | `runId` | `String` | no | Unique run id. |
| | `datasetRef` | `String` | no | Dataset fixture identifier. |
| | `objective` | `String` | no | ML objective. |
| | `status` | `RunStatus` | no | See enum. |
| | `ledger` | `Optional<PipelineLedger>` | yes | Populated after `RunPlanned`. |
| | `evalLedger` | `Optional<EvaluationLedger>` | yes | Populated after first `StageRecorded` or `StageGateFailed`. |
| | `report` | `Optional<ModelReport>` | yes | Populated after `RunCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `RunFailed` / `RunFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `RunHaltedOperator` / `RunHaltedAutomatic`. |
| | `driftAlerts` | `List<DriftAlert>` | no | Appended by `DriftAlertRaised` (initially empty). |
| | `createdAt` | `Instant` | no | When `RunCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStage` | (sealed interface) | — | — | Permits `Continue(StageDispatch)`, `Replan(PipelineLedger revised)`, `Promote(ModelReport stub)`, `Fail(String reason)`. |

## Enums

- `SpecialistKind` → `DATA_PROFILER`, `FEATURE_ENGINEER`, `MODEL_TRAINER`, `MODEL_EVALUATOR`.
- `EvalVerdict` → `OK`, `GATE_FAILED`, `FAILED`, `BLOCKED`.
- `GateOutcome` → `PASSED`, `FAILED`, `NOT_APPLICABLE`.
- `RunStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `GATE_FAILED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`PipelineRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunCreated` | `runId, datasetRef, objective, createdAt` | → PLANNING |
| `RunPlanned` | `ledger` | → EXECUTING |
| `StageDispatched` | `dispatch` | no status change; sets `ledger.activeDispatch`. |
| `StageGateFailed` | `attempt, dispatch, blocker` | no status change; appends `EvalEntry` with `GATE_FAILED`. |
| `StageRecorded` | `entry: EvalEntry` | no status change; appends to `evalLedger.entries`. |
| `LedgerRevised` | `ledger: PipelineLedger` | no status change; replaces `ledger`. |
| `RunCompleted` | `report` | → COMPLETED, `finishedAt = now` |
| `RunFailed` | `failureReason` | → FAILED or GATE_FAILED, `finishedAt = now` |
| `RunHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `RunHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `RunFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |
| `DriftAlertRaised` | `alert: DriftAlert` | no status change; appends to `driftAlerts`. |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RunQueue`)

| Event | Payload |
|---|---|
| `RunSubmitted` | `runId, datasetRef, objective, requestedBy, submittedAt` |

## View row

`RunRow` mirrors `PipelineRun` minus the heavy evaluation payload — `evalLedger.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `content` (in the backing `StageResult`) is not embedded in the row. The UI fetches the full run by id on click via `GET /api/runs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6). `driftAlerts` is included in the row as a full list (alerts are compact; no truncation needed).
