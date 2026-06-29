# Data model — event-driven-planner-executor

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `JobRequest` | `prompt` | `String` | no | Operator-submitted automation request. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `JobLedger` | `facts` | `List<String>` | no | Facts the orchestrator believes are known. |
| | `missing` | `List<String>` | no | Facts the orchestrator believes are still needed. |
| | `plan` | `List<String>` | no | Ordered list of plan steps (3–8). |
| | `currentDispatch` | `Optional<DispatchDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `DispatchDecision` | `executor` | `ExecutorKind` | no | Which executor runs the step. |
| | `step` | `String` | no | One-sentence step description. |
| | `rationale` | `String` | no | One-sentence justification. |
| `StepResult` | `executor` | `ExecutorKind` | no | Executor that ran the step. |
| | `step` | `String` | no | Echo of the step. |
| | `ok` | `boolean` | no | True if the executor could fulfill the step. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `StepEntry` | `attempt` | `int` | no | 1-based attempt count for this `(executor, step)` pair. |
| | `executor` | `ExecutorKind` | no | Executor that ran (or would have run) this step. |
| | `step` | `String` | no | The step text. |
| | `verdict` | `StepVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `scrubbedResult` | `String` | no | Sanitized result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `StepLedger` | `entries` | `List<StepEntry>` | no | Append-only. |
| `JobReport` | `summary` | `String` | no | 60–120 word summary. |
| | `evidence` | `List<String>` | no | 3–5 cited bullets, each tagged by executor. |
| | `producedAt` | `Instant` | no | When the orchestrator produced the report. |
| `Job` (entity state) | `jobId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original prompt. |
| | `status` | `JobStatus` | no | See enum. |
| | `ledger` | `Optional<JobLedger>` | yes | Populated after `JobPlanned`. |
| | `steps` | `Optional<StepLedger>` | yes | Populated after first `StepRecorded` or `StepBlocked`. |
| | `report` | `Optional<JobReport>` | yes | Populated after `JobCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `JobFailed` / `JobFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `JobHaltedOperator`. |
| | `createdAt` | `Instant` | no | When `JobCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the job reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(DispatchDecision)`, `Replan(JobLedger revised)`, `Complete(JobReport stub)`, `Fail(String reason)`. |

## Enums

- `ExecutorKind` → `HTTP`, `QUEUE`, `DB`, `SCRIPT`.
- `StepVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `JobStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`JobEntity`)

| Event | Payload | Transition |
|---|---|---|
| `JobCreated` | `jobId, prompt, createdAt` | → PLANNING |
| `JobPlanned` | `ledger` | → EXECUTING |
| `StepDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `StepBlocked` | `attempt, dispatch, blocker` | no status change; appends a `StepEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `StepRecorded` | `entry: StepEntry` | no status change; appends to `steps.entries`. |
| `LedgerRevised` | `ledger: JobLedger` | no status change; replaces `ledger`. |
| `JobCompleted` | `report` | → COMPLETED, `finishedAt = now` |
| `JobFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `JobHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `JobFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `JobSubmitted` | `jobId, prompt, requestedBy, submittedAt` |

## View row

`JobRow` mirrors `Job` minus the heavy step payload — `steps.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full job by id on click via `GET /api/jobs/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
