# Architecture — workflow-orchestration

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `WorkflowRunEndpoint` accepts a `{workflowId}` POST, writes `RunCreated` onto `WorkflowRunEntity`, and starts `WorkflowRunPipeline` keyed by `"pipeline-" + runId`. The pipeline's first stage (`validateStage`) emits `ValidateStarted`, then calls `PipelineOrchestrationAgent` with `TaskDef.taskType(VALIDATE_STEPS)` and a `stage = VALIDATE` metadata tag. The agent invokes `ValidateTools.checkStepDependencies` and `ValidateTools.verifyStepPreconditions`; every call passes through `StageGuardrail` first. Once the agent returns a `ValidationReport`, the pipeline writes `StepsValidated` onto the entity and advances to `executeStage` — same pattern, the EXECUTE task carries `stage = EXECUTE`. Then `notifyStage` runs with `stage = NOTIFY`. After `NotificationSent` lands, `evalStage` runs `StepCoverageScorer` over the recorded `(ValidationReport, ExecutionResult, RunResult)` triple — no LLM call — and writes `RunEvaluated`. `WorkflowRunView` projects every event into a read-model row; `WorkflowRunEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `StepCoverageScorer` is a deterministic rule-based scorer; that is what makes this a faithful **single-agent sequential-pipeline** example.

## Interaction sequence

The sequence traces the happy path (J1). Two properties are worth pausing on:

1. The stage boundary IS the dependency contract. Between `validateStage` and `executeStage`, the pipeline writes `StepsValidated` onto the entity. The next stage then reads `validation` from the entity to build the EXECUTE task's instruction context. The agent never sees validate-stage context inside the execute task's conversation; the typed handoff is the only path information travels.
2. Every tool call is filtered through `StageGuardrail`. The guardrail reads the in-flight task's `stage` metadata and the current `WorkflowRunEntity.status`, and applies the per-stage accept matrix. A misordered call is rejected before the tool body executes.

The agent calls themselves are bounded by per-stage timeouts (60 s on validate / execute / notify). `evalStage` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → VALIDATING → VALIDATED → EXECUTING → EXECUTED → NOTIFYING → NOTIFIED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `VALIDATING`, `EXECUTING`, or `NOTIFYING`. A `FAILED` run's prior data is preserved on the entity — the UI shows the partial state.
- `StageGuardrailRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a stage timeout transitions to `FAILED`.

There is no `APPROVED` or `DEPLOYED` state. The run result is informational; the operator reviews it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`WorkflowRunEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the guardrail audit, the failure, and the initial creation. `WorkflowRunView` projects every event into a row used by the UI. `WorkflowRunPipeline` both reads (`getRun`) and writes (`startValidate`, `recordValidation`, `startExecute`, `recordExecution`, `startNotify`, `recordResult`, `recordEvaluation`, `recordGuardrailRejection`, `fail`) on the entity. The relationship between `PipelineOrchestrationAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any run that lands in the entity log, the workflow definition passed through:

1. **Stage-gate guardrail** — every tool call is filtered. A NOTIFY-stage tool called during VALIDATE is rejected before the tool body runs; a `StageGuardrailRejected` event records the violation for audit.
2. **PipelineOrchestrationAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next stage.
3. **On-completion evaluator** — every emitted run result gets a 1–5 step-coverage score. Step coverage, phantom-step check, valid-outcome check, and count parity are each worth one point on a base of 1.

Each step is independent. The guardrail does not check step coverage; the evaluator does not check stage order. Removing one of them opens an explicit gap the other does not silently cover.
