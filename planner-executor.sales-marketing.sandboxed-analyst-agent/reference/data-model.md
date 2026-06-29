# Data model — sandboxed-analyst-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `DatasetUpload` | `datasetName` | `String` | no | Name of the uploaded CSV file. |
| | `uploaderEmail` | `String` | no | Submitter's email address. |
| | `csvContent` | `String` | no | Raw CSV text (may be empty if using a fixture path). |
| `AnalysisLedger` | `datasetFacts` | `List<String>` | no | Facts the analyst believes are known about the dataset. |
| | `hypotheses` | `List<String>` | no | Hypotheses the analyst intends to test. |
| | `plan` | `List<String>` | no | Ordered list of plan steps (3–8). Each step names a `ScriptKind`. |
| | `currentStep` | `Optional<StepDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `StepDecision` | `scriptKind` | `ScriptKind` | no | Category of analysis this script performs. |
| | `pythonScript` | `String` | no | Python snippet to run in the sandbox. |
| | `rationale` | `String` | no | One-sentence justification. |
| `ExecutionResult` | `scriptKind` | `ScriptKind` | no | Script kind executed. |
| | `script` | `String` | no | Echo of the script that ran. |
| | `ok` | `boolean` | no | True if the script ran without a Python exception. |
| | `stdout` | `String` | no | Raw captured standard output (pre-sanitize). |
| | `stderr` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ProgressEntry` | `attempt` | `int` | no | 1-based attempt count for this `(scriptKind, script)` pair. |
| | `scriptKind` | `ScriptKind` | no | Script kind that ran (or would have run). |
| | `script` | `String` | no | The script text. |
| | `verdict` | `ProgressVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `scrubbedOutput` | `String` | no | Sanitized stdout. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ProgressLedger` | `entries` | `List<ProgressEntry>` | no | Append-only. |
| `AnalysisFinding` | `metricName` | `String` | no | Name of the metric or insight. |
| | `value` | `String` | no | Computed value or observation. |
| | `interpretation` | `String` | no | One-sentence explanation citing the step that produced it. |
| `AnalysisReport` | `summary` | `String` | no | 60–120 word narrative summary. |
| | `findings` | `List<AnalysisFinding>` | no | 2–5 structured findings. |
| | `charts` | `List<String>` | no | Names of charts produced (paths relative to `/workspace/`). |
| | `producedAt` | `Instant` | no | When the analyst produced the report. |
| `AnalysisJob` (entity state) | `jobId` | `String` | no | Unique id. |
| | `datasetName` | `String` | no | Original dataset name. |
| | `uploaderEmail` | `String` | no | Submitter's email. |
| | `status` | `JobStatus` | no | See enum. |
| | `ledger` | `Optional<AnalysisLedger>` | yes | Populated after `JobPlanned`. |
| | `progress` | `Optional<ProgressLedger>` | yes | Populated after first `StepRecorded` or `StepBlocked`. |
| | `report` | `Optional<AnalysisReport>` | yes | Populated after `JobCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `JobFailed` / `JobFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `JobHaltedOperator` / `JobHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `JobCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(StepDecision)`, `Replan(AnalysisLedger revised)`, `Complete(AnalysisReport stub)`, `Fail(String reason)`. |

## Enums

- `ScriptKind` → `LOAD_INSPECT`, `AGGREGATE`, `SENTIMENT`, `SEGMENT`, `VISUALISE`, `SUMMARISE`.
- `ProgressVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `JobStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`AnalysisJobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, datasetName, uploaderEmail, createdAt` | → PLANNING |
| `JobPlanned` | `ledger` | → EXECUTING |
| `StepDispatched` | `stepDecision` | no status change; sets `ledger.currentStep`. |
| `StepBlocked` | `attempt, stepDecision, blocker` | no status change; appends a `ProgressEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `StepRecorded` | `entry: ProgressEntry` | no status change; appends to `progress.entries`. |
| `LedgerRevised` | `ledger: AnalysisLedger` | no status change; replaces `ledger`. |
| `JobCompleted` | `report` | → COMPLETED, `finishedAt = now` |
| `JobFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `JobHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`UploadQueue`)

| Event | Payload |
|---|---|
| `DatasetSubmitted` | `jobId, datasetName, uploaderEmail, submittedAt` |

## View row

`AnalysisJobRow` mirrors `AnalysisJob` minus the heavy progress payload — `progress.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedOutput` is capped at 240 characters. The UI fetches the full job by id on click via `GET /api/jobs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
