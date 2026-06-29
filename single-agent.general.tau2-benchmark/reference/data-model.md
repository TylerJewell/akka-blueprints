# Data model — tau2-benchmark-agent

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `BenchmarkTask` | `taskId` | `String` | no | Stable id from the task corpus (e.g. `tau2-webnav-001`). |
| | `category` | `String` | no | `"web-navigation"`, `"tool-use"`, or `"multi-step-reasoning"`. |
| | `description` | `String` | no | Plain-language statement of what the task asks. |
| | `steps` | `List<String>` | no | Ordered list of step descriptions (1–N). |
| | `referenceAnswer` | `String` | no | Expected final answer; used by `TaskScorer` for correctness check. |
| | `maxSteps` | `int` | no | Upper bound on the number of steps the agent may take. |
| `RunRequest` | `runId` | `String` | no | UUID minted by `BenchmarkEndpoint`. |
| | `task` | `BenchmarkTask` | no | The submitted task definition. |
| | `submittedBy` | `String` | no | User identifier. |
| | `submittedAt` | `Instant` | no | When the endpoint received the request. |
| `StepOutput` | `stepIndex` | `int` | no | 0-based index matching `task.steps[stepIndex]`. |
| | `output` | `String` | no | What the agent found or produced for this step. |
| | `attempted` | `boolean` | no | `true` if a genuine attempt was made; `false` if skipped. |
| `TaskResult` | `outcome` | `Outcome` | no | Enum value. |
| | `stepOutputs` | `List<StepOutput>` | no | One entry per step. |
| | `finalAnswer` | `String` | no | The agent's answer to the overall task. |
| | `latencyMs` | `long` | no | Elapsed ms from task start to completion. |
| | `completedAt` | `Instant` | no | When the agent returned. |
| `PerformanceScore` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence. |
| | `scoredAt` | `Instant` | no | When `TaskScorer` finished. |
| `BenchmarkRun` (entity state) | `runId` | `String` | no | — |
| | `request` | `Optional<RunRequest>` | yes | Populated after `RunSubmitted`. |
| | `result` | `Optional<TaskResult>` | yes | Populated after `ResultRecorded`. |
| | `score` | `Optional<PerformanceScore>` | yes | Populated after `RunScored`. |
| | `status` | `RunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunSubmitted` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |

Every nullable field on `BenchmarkRun` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`Outcome`: `PASS`, `PARTIAL`, `FAIL`.
`RunStatus`: `SUBMITTED`, `EXECUTING`, `RESULT_RECORDED`, `SCORED`, `FAILED`.

## Events (`BenchmarkRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunSubmitted` | `request` | → SUBMITTED |
| `ExecutionStarted` | — | → EXECUTING |
| `ResultRecorded` | `result` | → RESULT_RECORDED |
| `RunScored` | `score` | → SCORED (terminal happy) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `BenchmarkRun.initial("")` with all `Optional` fields as `Optional.empty()` and `status = SUBMITTED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`RunRow` mirrors `BenchmarkRun` in full — there is no audit-only field that must be omitted from the view. The task definition is included in `request.task` so the UI can display steps in the detail pane without a separate entity fetch.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM benchmark_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side. The aggregate stats (`GET /api/runs/aggregate`) are computed server-side from the full list returned by this query.

## Task definition (`BenchmarkTasks.java`)

```java
public final class BenchmarkTasks {
  public static final Task<TaskResult> EXECUTE_TASK = Task
      .name("Execute benchmark task")
      .description("Execute the given benchmark task steps and produce a TaskResult")
      .resultConformsTo(TaskResult.class);

  private BenchmarkTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Scoring rubric (`TaskScorer.java`)

Four criteria, each worth one point (base score = 1):

| Criterion | Check | Points if met |
|---|---|---|
| Step coverage | `stepOutputs.size() == task.steps().size()` | +1 |
| Step content | all `StepOutput.output` are non-empty | +1 |
| Answer correctness | `finalAnswer.trim().equalsIgnoreCase(task.referenceAnswer().trim())` | +1 |
| Latency bound | `latencyMs > 0 && latencyMs <= 120_000` | +1 |

Score range: 1 (no criteria met) to 5 (all four criteria met). Rationale is a single sentence naming which criteria passed and which failed.
