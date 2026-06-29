# Data model — orchestrator-web-file-code

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskRequest` | `prompt` | `String` | no | User-submitted task prompt. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `TaskLedger` | `facts` | `List<String>` | no | Facts the orchestrator believes are known. |
| | `missing` | `List<String>` | no | Facts the orchestrator believes are still needed. |
| | `plan` | `List<String>` | no | Ordered list of plan steps (3–8). |
| | `currentDispatch` | `Optional<DispatchDecision>` | yes | Populated between `proposeStep` and `recordStep`; cleared at end-of-loop. |
| `DispatchDecision` | `specialist` | `SpecialistKind` | no | Which specialist runs the subtask. |
| | `subtask` | `String` | no | One-sentence subtask description. |
| | `rationale` | `String` | no | One-sentence justification. |
| `SubtaskResult` | `specialist` | `SpecialistKind` | no | Specialist that ran the subtask. |
| | `subtask` | `String` | no | Echo of the subtask. |
| | `ok` | `boolean` | no | True if the specialist could fulfill the subtask. |
| | `content` | `String` | no | Raw textual result (pre-sanitize). |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `ProgressEntry` | `attempt` | `int` | no | 1-based attempt count for this `(specialist, subtask)` pair. |
| | `specialist` | `SpecialistKind` | no | Specialist that ran (or would have run) this subtask. |
| | `subtask` | `String` | no | The subtask text. |
| | `verdict` | `ProgressVerdict` | no | OK / BLOCKED_BY_GUARDRAIL / FAILED / UNSAFE. |
| | `scrubbedResult` | `String` | no | Sanitized result text. |
| | `blocker` | `Optional<String>` | yes | Populated on BLOCKED or FAILED. |
| | `recordedAt` | `Instant` | no | When the entry was appended. |
| `ProgressLedger` | `entries` | `List<ProgressEntry>` | no | Append-only. |
| `TaskAnswer` | `summary` | `String` | no | 60–120 word summary. |
| | `evidence` | `List<String>` | no | 3–5 cited bullets, each tagged by specialist. |
| | `producedAt` | `Instant` | no | When the orchestrator produced the answer. |
| `Task` (entity state) | `taskId` | `String` | no | Unique id. |
| | `prompt` | `String` | no | Original prompt. |
| | `status` | `TaskStatus` | no | See enum. |
| | `ledger` | `Optional<TaskLedger>` | yes | Populated after `TaskPlanned`. |
| | `progress` | `Optional<ProgressLedger>` | yes | Populated after first `SubtaskRecorded` or `SubtaskBlocked`. |
| | `answer` | `Optional<TaskAnswer>` | yes | Populated after `TaskCompleted`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `TaskFailed` / `TaskFailedTimeout`. |
| | `haltReason` | `Optional<String>` | yes | Populated on `TaskHaltedOperator` / `TaskHaltedAutomatic`. |
| | `createdAt` | `Instant` | no | When `TaskCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the task reached a terminal state. |
| `SystemControl` (entity state) | `halted` | `boolean` | no | Operator halt flag. |
| | `reason` | `Optional<String>` | yes | Set when `HaltRequested`. |
| | `haltedAt` | `Optional<Instant>` | yes | Set when `HaltRequested`; cleared when `HaltCleared`. |
| `NextStep` | (sealed interface) | — | — | Permits `Continue(DispatchDecision)`, `Replan(TaskLedger revised)`, `Complete(TaskAnswer stub)`, `Fail(String reason)`. |

## Enums

- `SpecialistKind` → `WEB`, `FILE`, `CODER`, `TERMINAL`.
- `ProgressVerdict` → `OK`, `BLOCKED_BY_GUARDRAIL`, `FAILED`, `UNSAFE`.
- `TaskStatus` → `PLANNING`, `EXECUTING`, `COMPLETED`, `FAILED`, `HALTED`, `STUCK`.

## Events (`TaskEntity`)

| Event | Payload | Transition |
|---|---|---|
| `TaskCreated` | `taskId, prompt, createdAt` | → PLANNING |
| `TaskPlanned` | `ledger` | → EXECUTING |
| `SubtaskDispatched` | `dispatch` | no status change; sets `ledger.currentDispatch`. |
| `SubtaskBlocked` | `attempt, dispatch, blocker` | no status change; appends a `ProgressEntry` with verdict `BLOCKED_BY_GUARDRAIL`. |
| `SubtaskRecorded` | `entry: ProgressEntry` | no status change; appends to `progress.entries`. |
| `LedgerRevised` | `ledger: TaskLedger` | no status change; replaces `ledger`. |
| `TaskCompleted` | `answer` | → COMPLETED, `finishedAt = now` |
| `TaskFailed` | `failureReason` | → FAILED, `finishedAt = now` |
| `TaskHaltedAutomatic` | `haltReason` | → HALTED, `finishedAt = now` |
| `TaskHaltedOperator` | `haltReason` | → HALTED, `finishedAt = now` |
| `TaskFailedTimeout` | `failureReason` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `HaltRequested` | `reason, haltedAt` |
| `HaltCleared` | `clearedAt` |

## Events (`RequestQueue`)

| Event | Payload |
|---|---|
| `TaskSubmitted` | `taskId, prompt, requestedBy, submittedAt` |

## View row

`TaskRow` mirrors `Task` minus the heavy progress payload — `progress.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `scrubbedResult` is capped at 240 characters. The UI fetches the full task by id on click via `GET /api/tasks/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
