# Data model — agent-benchmark-harness

Authoritative; `/akka:implement` produces these records and events verbatim. Nullable lifecycle fields are `Optional<T>` (Lesson 6).

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `EvalTask` | `taskId` | `String` | no | Unique identifier for the evaluation task. |
| | `prompt` | `String` | no | The prompt submitted to the target model. |
| | `referenceAnswer` | `String` | no | The authoritative correct answer or criteria. |
| | `category` | `String` | no | `reasoning`, `factual`, or `instruction-following`. |
| `TaskResponse` | `taskId` | `String` | no | Echoed from the task input. |
| | `rawOutput` | `String` | no | The target model's verbatim response. |
| | `respondedAt` | `Instant` | no | When the runner received the response. |
| `ScoredResult` | `taskId` | `String` | no | Echoed from the task input. |
| | `verdict` | `TaskVerdict` | no | `PASS` or `FAIL`. |
| | `score` | `int` | no | 0–100 numeric score. |
| | `rationale` | `String` | no | One-sentence explanation of the verdict. |
| | `scoredAt` | `Instant` | no | When the scorer returned the result. |
| `TaskAttempt` | `taskId` | `String` | no | Task identifier. |
| | `response` | `TaskResponse` | no | The runner's output for this task. |
| | `scored` | `ScoredResult` | no | The scorer's grade for this task. |
| `RunAggregate` | `totalTasks` | `int` | no | Number of tasks in the run. |
| | `passCount` | `int` | no | Tasks with verdict PASS. |
| | `failCount` | `int` | no | Tasks with verdict FAIL. |
| | `passRate` | `double` | no | `passCount / totalTasks`. |
| | `durationMs` | `long` | no | Wall-clock duration from RunStarted to terminal event. |
| `BenchmarkRun` (entity state) | `runId` | `String` | no | Unique run identifier. |
| | `triggeredBy` | `String` | no | Who or what triggered the run. |
| | `status` | `RunStatus` | no | See enum. |
| | `taskAttempts` | `List<TaskAttempt>` | no | Appended as tasks complete; bounded at `maxTasksPerRun`. |
| | `aggregate` | `Optional<RunAggregate>` | yes | Populated on RunSummaryEvent. |
| | `failureReason` | `Optional<String>` | yes | Populated on FAILED terminal transition. |
| | `startedAt` | `Instant` | no | When RunStarted emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | When the run reached a terminal state. |

## Enums

`RunStatus`: `PENDING`, `RUNNING`, `PASSED`, `FAILED`.

`TaskVerdict`: `PASS`, `FAIL`.

## Events (`RunEntity`)

| Event | Payload | Trigger | Transition |
|---|---|---|---|
| `RunStarted` | `runId, triggeredBy, totalTasks, threshold, startedAt` | Workflow `initStep` | `PENDING` → `RUNNING` |
| `TaskResponseRecorded` | `taskId, response: TaskResponse` | After `runStep` returns | (no status change; appends partial `TaskAttempt` to `taskAttempts[]`) |
| `TaskScoredEvent` | `taskId, scored: ScoredResult` | After `scoreStep` returns | (no status change; completes the `TaskAttempt` for `taskId` in `taskAttempts[]`) |
| `RunSummaryEvent` | `aggregate: RunAggregate, status, failureReason?` | After `aggregateStep` determines pass/fail | → `PASSED` or `FAILED`, `finishedAt = now` |
| `RunAborted` | `failureReason, abortedAt` | `defaultStepRecovery` failover after exhausting retries | → `FAILED`, `finishedAt = now` |
| `AccuracySnapshotRecorded` | `passRate, passCount, failCount, recordedAt` | `AccuracySampler` per completed run; also emitted inline with `RunSummaryEvent` | (no status change; visible in the run's event history) |

## Events (`TaskRegistry`)

| Event | Payload |
|---|---|
| `TaskRegistered` | `taskId, prompt, referenceAnswer, category, registeredAt` |

## View row

`RunRow` is structurally identical to `BenchmarkRun` — the `taskAttempts` list is bounded at `maxTasksPerRun` (default 50) so the row stays small enough to push down the SSE stream without pagination. The view's `TableUpdater` consumes every `RunEntity` event and rewrites the row atomically.

Because Akka cannot auto-index enum columns (Lesson 2), the view exposes **one** query: `getAllRuns` returning the full list. Callers filter by `status` client-side.

`GET /api/ci-gate` reads the same `getAllRuns` result and selects the most recent run by `finishedAt` where `status` is `PASSED` or `FAILED`. The threshold used in the comparison is the one stored on the run's `RunStarted` event, not the current configuration value.
