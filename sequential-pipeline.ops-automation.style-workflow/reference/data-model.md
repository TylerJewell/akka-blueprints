# Data model — workflow-orchestration

Authoritative; `/akka:implement` produces these records and events verbatim.

## Records

| Record | Field | Type | Nullable | Meaning |
|---|---|---|---|---|
| `StepDefinition` | `stepId` | `String` | no | Unique step identifier within the workflow. |
| | `name` | `String` | no | Human-readable step name. |
| | `dependsOn` | `List<String>` | no | Possibly empty; each entry is a `stepId` that must complete first. |
| | `command` | `String` | no | Command or operation the step performs. |
| `WorkflowDefinition` | `workflowId` | `String` | no | Unique workflow identifier. |
| | `name` | `String` | no | Human-readable workflow name. |
| | `steps` | `List<StepDefinition>` | no | Possibly empty. |
| `StepValidation` | `stepId` | `String` | no | MUST equal a `StepDefinition.stepId` from the workflow. |
| | `status` | `String` | no | `OK` or `WARN`. |
| | `warning` | `String` | no | Empty string when `status == OK`. |
| `ValidationReport` | `workflowId` | `String` | no | The workflow being validated. |
| | `validations` | `List<StepValidation>` | no | Possibly empty; J6 demonstrates the empty path. |
| | `allPassed` | `boolean` | no | True when no validation has `status == WARN` that blocks execution. |
| | `validatedAt` | `Instant` | no | When the VALIDATE task returned. |
| `StepOutcome` | `stepId` | `String` | no | MUST equal a `StepDefinition.stepId` from the upstream `ValidationReport`. |
| | `status` | `String` | no | `SUCCEEDED`, `FAILED`, or `SKIPPED`. |
| | `durationMs` | `long` | no | Execution duration in milliseconds. |
| | `detail` | `String` | no | Empty string when no detail applies. |
| `ExecutionResult` | `workflowId` | `String` | no | The workflow being executed. |
| | `outcomes` | `List<StepOutcome>` | no | Possibly empty. |
| | `executedAt` | `Instant` | no | When the EXECUTE task returned. |
| `NotificationReceipt` | `channel` | `String` | no | Notification channel (e.g. `ops-alerts`). |
| | `messageId` | `String` | no | Stable id minted by `dispatchNotification`. |
| | `sentAt` | `Instant` | no | When the notification was dispatched. |
| `RunSummary` | `workflowId` | `String` | no | The workflow summarised. |
| | `totalSteps` | `int` | no | Total number of steps executed. |
| | `succeeded` | `int` | no | Count of `SUCCEEDED` outcomes. |
| | `failed` | `int` | no | Count of `FAILED` outcomes. |
| | `skipped` | `int` | no | Count of `SKIPPED` outcomes. |
| `RunResult` | `runId` | `String` | no | The run identifier. |
| | `workflowId` | `String` | no | The workflow that was run. |
| | `summary` | `RunSummary` | no | Aggregated totals. |
| | `stepOutcomes` | `List<StepOutcome>` | no | Full outcome list mirroring `ExecutionResult.outcomes`. |
| | `notification` | `NotificationReceipt` | no | Dispatch receipt. |
| | `completedAt` | `Instant` | no | When the NOTIFY task returned. |
| `EvalResult` | `score` | `int` | no | 1–5. |
| | `rationale` | `String` | no | One sentence naming the largest gap (or "all checks passed"). |
| | `evaluatedAt` | `Instant` | no | When `StepCoverageScorer` finished. |
| `GuardrailRejection` | `stage` | `String` | no | `VALIDATE` / `EXECUTE` / `NOTIFY`. |
| | `tool` | `String` | no | Name of the misordered tool. |
| | `reason` | `String` | no | Structured reason from `StageGuardrail`. |
| | `rejectedAt` | `Instant` | no | When the guardrail fired. |
| `WorkflowRunRecord` (entity state) | `runId` | `String` | no | — |
| | `workflowId` | `Optional<String>` | yes | Populated after `RunCreated`. |
| | `validation` | `Optional<ValidationReport>` | yes | Populated after `StepsValidated`. |
| | `execution` | `Optional<ExecutionResult>` | yes | Populated after `StepsExecuted`. |
| | `result` | `Optional<RunResult>` | yes | Populated after `NotificationSent`. |
| | `eval` | `Optional<EvalResult>` | yes | Populated after `RunEvaluated`. |
| | `status` | `WorkflowRunStatus` | no | See enum. |
| | `createdAt` | `Instant` | no | When `RunCreated` emitted. |
| | `finishedAt` | `Optional<Instant>` | yes | Terminal-state timestamp. |
| | `guardrailRejections` | `List<GuardrailRejection>` | no | Appended on every `StageGuardrailRejected` event; empty on the happy path. |

Every nullable lifecycle field on `WorkflowRunRecord` is `Optional<T>`. The view's table updater wraps values with `Optional.of(...)`; callers use `.orElse(...)` or `.isPresent()` (Lesson 6).

## Enums

`WorkflowRunStatus`: `CREATED`, `VALIDATING`, `VALIDATED`, `EXECUTING`, `EXECUTED`, `NOTIFYING`, `NOTIFIED`, `EVALUATED`, `FAILED`.

`Stage` (used by `@FunctionTool` annotations and `StageGuardrail`): `VALIDATE`, `EXECUTE`, `NOTIFY`.

## Events (`WorkflowRunEntity`)

| Event | Payload | Transition |
|---|---|---|
| `RunCreated` | `workflowId: String` | → CREATED |
| `ValidateStarted` | — | → VALIDATING |
| `StepsValidated` | `validation: ValidationReport` | → VALIDATED |
| `ExecuteStarted` | — | → EXECUTING |
| `StepsExecuted` | `execution: ExecutionResult` | → EXECUTED |
| `NotifyStarted` | — | → NOTIFYING |
| `NotificationSent` | `result: RunResult` | → NOTIFIED |
| `RunEvaluated` | `eval: EvalResult` | → EVALUATED (terminal happy) |
| `StageGuardrailRejected` | `stage, tool, reason, rejectedAt` | no status change (audit-only) |
| `RunFailed` | `reason: String` | → FAILED (terminal) |

`emptyState()` returns `WorkflowRunRecord.initial("")` with all `Optional` fields as `Optional.empty()`, `guardrailRejections = List.of()`, and `status = CREATED`. `emptyState()` never references `commandContext()` (Lesson 3).

## View row

`WorkflowRunRow` mirrors `WorkflowRunRecord` exactly. The UI fetches the full row via `GET /api/runs/{id}` and streams updates via `GET /api/runs/sse`.

The view declares ONE query: `getAllRuns: SELECT * AS runs FROM workflow_run_view`. No `WHERE status = :status` filter — Akka cannot auto-index enum columns (Lesson 2); the endpoint filters client-side.

## Task definitions (`WorkflowTasks.java`)

```java
public final class WorkflowTasks {
  public static final Task<ValidationReport> VALIDATE_STEPS = Task
      .name("Validate steps")
      .description("Check step dependency graph and verify preconditions for all declared steps")
      .resultConformsTo(ValidationReport.class);

  public static final Task<ExecutionResult> EXECUTE_STEPS = Task
      .name("Execute steps")
      .description("Run each step in dependency order and record outcomes")
      .resultConformsTo(ExecutionResult.class);

  public static final Task<RunResult> NOTIFY_COMPLETION = Task
      .name("Notify completion")
      .description("Build run summary and dispatch completion notification")
      .resultConformsTo(RunResult.class);

  private WorkflowTasks() {}
}
```

The companion `Tasks` class is mandatory for any `AutonomousAgent` (Lesson 7).

## Stage-tagged tools

Each `@FunctionTool` method on `ValidateTools`, `ExecuteTools`, and `NotifyTools` carries a `Stage` constant. `StageGuardrail` reads this constant before the tool body runs and rejects calls whose stage does not match the per-status accept matrix (see eval-matrix.yaml H1). The tool registry is built once at startup; the guardrail reads it for every call.
