# Data model — trajectory-eval

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `TaskScenario` | `scenarioId` | `String` | no | Identifier of the task scenario to evaluate. |
| | `taskDescription` | `String` | no | Human-readable description of what the task requires. |
| | `requestedBy` | `String` | no | UI identifier of the submitter. |
| `ToolCall` | `toolName` | `String` | no | Name of the tool called. |
| | `inputs` | `Map<String, Object>` | no | Parameter map passed to the tool. |
| | `output` | `String` | no | Tool's return value serialised as a string. |
| | `calledAt` | `Instant` | no | When the Runner invoked the tool. |
| `RecordedTrajectory` | `steps` | `List<ToolCall>` | no | Ordered list of tool calls; empty before any run. |
| | `stepCount` | `int` | no | Length of `steps`. |
| | `recordedAt` | `Instant` | no | When the Runner returned the trajectory. |
| `GuardrailVerdict` | `passed` | `boolean` | no | Whether the guardrail accepted the trajectory. |
| | `reasonCode` | `String` | no | `OK`, `OVER_STEP_LIMIT`, or a future-added code. |
| | `detail` | `String` | no | Human-readable detail; empty when `passed=true`. |
| `Deviation` | `stepIndex` | `int` | no | 0-indexed position in the trajectory where divergence occurred. |
| | `expectedTool` | `String` | no | Tool name the reference path requires at this step. |
| | `actualTool` | `String` | no | Tool name the Runner called at this step. |
| | `note` | `String` | no | One sentence describing the mismatch. |
| `DeviationReport` | `deviations` | `List<Deviation>` | no | 0-or-more deviations; empty on `PASS`. |
| | `overallRationale` | `String` | no | One-sentence summary; required either way. |
| `TrajectoryVerdict` | `verdict` | `EvalVerdict` | no | `PASS` or `FAIL`. |
| | `report` | `DeviationReport` | no | See above. |
| | `deviationCount` | `int` | no | Length of `report.deviations`. |
| | `evaluatedAt` | `Instant` | no | When the Evaluator returned. |
| `TrajectoryAttempt` | `attemptNumber` | `int` | no | 1-indexed; monotonic across the loop (includes guardrail-blocked attempts). |
| | `trajectory` | `RecordedTrajectory` | no | The Runner's output for this attempt. |
| | `guardrail` | `GuardrailVerdict` | no | The deterministic check's verdict. |
| | `verdict` | `Optional<TrajectoryVerdict>` | yes | Empty when the guardrail blocked; populated otherwise. |
| `ReferencePath` | `scenarioId` | `String` | no | Scenario this path belongs to. |
| | `steps` | `List<ToolCall>` | no | Ordered reference tool calls (only `toolName` is compared). |
| | `registeredAt` | `Instant` | no | When `ReferenceRegistered` was emitted. |
| | `updatedAt` | `Optional<Instant>` | yes | Populated when `ReferenceUpdated` is emitted. |
| `Evaluation` (entity state) | `evaluationId` | `String` | no | Unique id. |
| | `scenarioId` | `String` | no | The scenario under evaluation. |
| | `taskDescription` | `String` | no | Task description resolved at submission. |
| | `maxAttempts` | `int` | no | Per-evaluation retry ceiling (default 4). |
| | `status` | `EvaluationStatus` | no | See enum. |
| | `attempts` | `List<TrajectoryAttempt>` | no | Bounded at `maxAttempts`; starts empty. |
| | `passedAttemptNumber` | `Optional<Integer>` | yes | Populated on `EvaluationPassed`. |
| | `passedTrajectory` | `Optional<RecordedTrajectory>` | yes | Populated on `EvaluationPassed`. |
| | `failureReason` | `Optional<String>` | yes | Populated on `EvaluationFailedFinal`. |
| | `createdAt` | `Instant` | no | When `EvaluationCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the evaluation reached a terminal state. |

## Enums

`EvaluationStatus`: `RUNNING`, `EVALUATING`, `PASSED`, `FAILED_FINAL`.

`EvalVerdict`: `PASS`, `FAIL`.

## Events (`EvaluationEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `EvaluationCreated` | `evaluationId, scenarioId, taskDescription, maxAttempts, createdAt` | Workflow `startStep` | → `RUNNING` |
| `TrajectoryRecorded` | `attemptNumber, trajectory: RecordedTrajectory` | After `executeStep` returns | (no status change; appends to `attempts[]`) |
| `TrajectoryGuardrailVerdictRecorded` | `attemptNumber, verdict: GuardrailVerdict` | After `guardrailStep` | `passed=true` → `EVALUATING`; `passed=false` → `RUNNING` (re-execute) |
| `TrajectoryEvaluated` | `attemptNumber, verdict: TrajectoryVerdict` | After `evaluateStep` returns | (no status change; populates `attempts[n].verdict`) |
| `EvaluationPassed` | `attemptNumber, passedTrajectory: RecordedTrajectory` | `TrajectoryVerdict.verdict = PASS` | → `PASSED`, `finishedAt = now` |
| `EvaluationFailedFinal` | `bestAttemptNumber, bestTrajectory: RecordedTrajectory, failureReason` | `attempts.size() == maxAttempts` AND last verdict is `FAIL`; OR `defaultStepRecovery` failover | → `FAILED_FINAL`, `finishedAt = now` |
| `EvalRecorded` | `attemptNumber, verdict, deviationCount, stepLimitExceeded, recordedAt` | `EvalSampler` per evaluated attempt; workflow on terminal transition | (no status change; appended to internal `evalEvents[]` projection) |

## Events (`ReferenceStore`)

| Event | Payload |
|---|---|
| `ReferenceRegistered` | `scenarioId, steps: List<ToolCall>, registeredAt` |
| `ReferenceUpdated` | `scenarioId, steps: List<ToolCall>, updatedAt` |

## View row

`EvaluationRow` is structurally identical to `Evaluation` — the `attempts` list is bounded at `maxAttempts` (default 4) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `EvaluationEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllEvaluations` returning the full list. Callers filter by `status` client-side.
