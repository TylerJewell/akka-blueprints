# Data model — plan-execute-replan

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `GoalRequest` | `goal` | `String` | no | User-submitted goal text. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ToolAssignment` | `kind` | `ToolKind` | no | Which tool the Executor invokes. |
| | `argument` | `String` | no | Concrete argument for that tool. |
| `PlanStep` | `stepIndex` | `int` | no | 0-based position in the plan. |
| | `description` | `String` | no | One-sentence step description. |
| | `tool` | `ToolAssignment` | no | Tool to invoke and argument to pass. |
| | `expectedOutput` | `String` | no | Short phrase naming the expected result. |
| `ExecutionPlan` | `goalSummary` | `String` | no | One-sentence restatement of the goal. |
| | `steps` | `List<PlanStep>` | no | Ordered steps (3–5). |
| `StepResult` | `stepIndex` | `int` | no | Step this result belongs to. |
| | `tool` | `ToolKind` | no | Tool that ran (echoed from `ToolAssignment`). |
| | `ok` | `boolean` | no | True if the Executor fulfilled the step. |
| | `output` | `String` | no | Raw textual output. |
| | `errorReason` | `Optional<String>` | yes | Populated when `ok=false`. |
| `EvalRecord` | `dimension` | `String` | no | Evaluation dimension label. |
| | `score` | `int` | no | 0–100 quality score. |
| | `rationale` | `String` | no | One-sentence justification. |
| `Observation` | `stepIndex` | `int` | no | Step this observation relates to. |
| | `type` | `ObservationType` | no | STEP_OK / STEP_FAILED / STEP_BLOCKED / PLAN_REVISED / EVAL. |
| | `content` | `String` | no | Step output, block reason, revision summary, or eval note. |
| | `evalRecord` | `Optional<EvalRecord>` | yes | Populated only when type is EVAL. |
| | `recordedAt` | `Instant` | no | When the observation was appended. |
| `ObservationLedger` | `entries` | `List<Observation>` | no | Append-only. |
| `GoalConclusion` | `summary` | `String` | no | 60–100 word summary. |
| | `citations` | `List<String>` | no | 3–5 bullets prefixed by tool kind. |
| | `producedAt` | `Instant` | no | When the Replanner produced the conclusion. |
| `Goal` (entity state) | `goalId` | `String` | no | Unique id. |
| | `goal` | `String` | no | Original goal text. |
| | `status` | `GoalStatus` | no | See enum. |
| | `plan` | `Optional<ExecutionPlan>` | yes | Populated after `GoalPlanned`. |
| | `currentStepIndex` | `int` | no | Index of the next step to execute. Starts at 0. |
| | `revisionCount` | `int` | no | Number of `Revise` decisions so far. |
| | `observations` | `Optional<ObservationLedger>` | yes | Populated after first observation. |
| | `conclusion` | `Optional<GoalConclusion>` | yes | Populated after `GoalConcluded`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `GoalFailed` / `GoalFailedTimeout`. |
| | `pauseReason` | `Optional<String>` | yes | Populated on `GoalPaused`. |
| | `createdAt` | `Instant` | no | When `GoalCreated` was emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the goal reached a terminal state. |
| `SystemControl` (entity state) | `paused` | `boolean` | no | Operator pause flag. |
| | `reason` | `Optional<String>` | yes | Set when `PauseRequested`. |
| | `pausedAt` | `Optional<Instant>` | yes | Set when `PauseRequested`; cleared when `PauseCleared`. |
| `ReplanDecision` | (sealed interface) | — | — | Permits `Continue(int nextStepIndex)`, `Revise(ExecutionPlan updatedPlan, String reason)`, `Conclude(GoalConclusion conclusion)`. |

## Enums

- `ToolKind` → `SEARCH`, `READ`, `CALCULATE`, `SUMMARISE`.
- `ObservationType` → `STEP_OK`, `STEP_FAILED`, `STEP_BLOCKED`, `PLAN_REVISED`, `EVAL`.
- `GoalStatus` → `PLANNING`, `EXECUTING`, `CONCLUDED`, `FAILED`, `PAUSED`, `STUCK`.

## Events (`GoalEntity`)

| Event | Payload | Transition |
|---|---|---|
| `GoalCreated` | `goalId, goal, createdAt` | → PLANNING |
| `GoalPlanned` | `plan: ExecutionPlan` | → EXECUTING |
| `StepDispatched` | `stepIndex, toolAssignment` | no status change |
| `StepBlocked` | `stepIndex, toolAssignment, reason` | no status change; appends STEP_BLOCKED Observation |
| `StepObserved` | `observation: Observation` | no status change; appends to `observations.entries` |
| `PlanRevised` | `updatedPlan: ExecutionPlan, revisionCount: int, reason: String` | no status change; replaces `plan`; appends PLAN_REVISED Observation |
| `EvalRecorded` | `evalRecord: EvalRecord` | no status change; appends EVAL Observation |
| `GoalConcluded` | `conclusion: GoalConclusion` | → CONCLUDED, `finishedAt = now` |
| `GoalFailed` | `failureReason: String` | → FAILED, `finishedAt = now` |
| `GoalPaused` | `pauseReason: String` | → PAUSED, `finishedAt = now` |
| `GoalFailedTimeout` | `failureReason: String` | → STUCK, `finishedAt = now` |

## Events (`SystemControlEntity`)

| Event | Payload |
|---|---|
| `PauseRequested` | `reason, pausedAt` |
| `PauseCleared` | `clearedAt` |

## Events (`GoalQueue`)

| Event | Payload |
|---|---|
| `GoalSubmitted` | `goalId, goal, requestedBy, submittedAt` |

## View row

`GoalRow` mirrors `Goal` minus the heavy observation payload — `observations.entries` is truncated to the last 3 entries plus a `truncatedFromTotal: int` count, and each entry's `content` is capped at 240 characters. The UI fetches the full goal by id on click via `GET /api/goals/{id}`. Every nullable field on the row record is declared `Optional<T>` (Lesson 6).
