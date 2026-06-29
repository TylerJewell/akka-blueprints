# SPEC — workflow-orchestration

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Workflows Process Orchestration.
**One-line pitch:** An operator submits a workflow run definition; one `PipelineOrchestrationAgent` walks it through three stage phases — **VALIDATE** declared steps, **EXECUTE** them in dependency order, **NOTIFY** on completion — with each stage gated on the prior stage's recorded output and each stage's tools rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in an ops-automation domain. One `PipelineOrchestrationAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit stage dependency**: the VALIDATE stage's typed output becomes the EXECUTE stage's instruction context; the EXECUTE stage's typed output becomes the NOTIFY stage's instruction context. The agent never holds all three stages in one conversation — each stage runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared stage (`VALIDATE` / `EXECUTE` / `NOTIFY`) and the current `WorkflowRunEntity` status. A NOTIFY-stage tool called while the entity has not yet recorded `StepsExecuted` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. The same hook enforces the deeper pipeline property: a stage's tool set is unreachable until its preconditions hold.
- An **`on-completion-eval`** runs immediately after `NotificationSent` lands, as `evalStage` inside the workflow. A deterministic, rule-based `StepCoverageScorer` (no LLM call — the eval is rule-based on purpose, so the same run always scores the same) checks that every declared step in the workflow definition has a matching execution record, that every execution record resolves to a known step id, that no execution was recorded for a step absent from the declaration, and that the step count equals the declared count.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the stage-boundary handoffs are the right cut to enforce both the dependency contract and the per-stage tool isolation.

## 3. User-facing flows

The user opens the App UI tab.

1. The operator picks a **workflow definition** from the input (or selects one of three seeded definitions — `database-migration-v3`, `service-deployment-canary`, `nightly-etl-refresh`).
2. The operator clicks **Run workflow**. The UI POSTs to `/api/runs` and receives a `runId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `VALIDATING` — the workflow has started `validateStage` and the agent has been handed the VALIDATE task.
4. Within ~10–20 s the card reaches `VALIDATED` — the typed `ValidationReport` is visible in the card detail (a table of declared steps with their validation status and any warnings). The agent's VALIDATE task returned; the workflow recorded `StepsValidated` and ran the EXECUTE task.
5. Within ~10–20 s more the card reaches `EXECUTING`. The `ValidationReport` step list is visible along with a live execution progress indicator.
6. Within ~10–20 s more the card reaches `EXECUTED`. The `ExecutionResult` is visible (per-step outcome list with status and duration). The agent's EXECUTE task returned; the workflow recorded `StepsExecuted`.
7. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `RunResult` — summary, per-step outcome, notification log — plus an eval score chip (1–5) and a one-line rationale.
8. The operator can submit another workflow definition; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `WorkflowRunEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `WorkflowRunEntity`, `WorkflowRunView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `WorkflowRunEntity` | `EventSourcedEntity` | Per-run lifecycle: created → validating → validated → executing → executed → notifying → notified → evaluated. Source of truth. | `WorkflowRunEndpoint`, `WorkflowRunPipeline` | `WorkflowRunView` |
| `WorkflowRunPipeline` | `Workflow` | One workflow per runId. Stages: `validateStage` → `executeStage` → `notifyStage` → `evalStage`. Each stage runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `WorkflowRunEndpoint` after `CREATED` | `PipelineOrchestrationAgent`, `WorkflowRunEntity` |
| `PipelineOrchestrationAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `WorkflowTasks.java`: `VALIDATE_STEPS` → `ValidationReport`, `EXECUTE_STEPS` → `ExecutionResult`, `NOTIFY_COMPLETION` → `RunResult`. Each task is registered with the stage-appropriate function tools. | invoked by `WorkflowRunPipeline` | returns typed results |
| `ValidateTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `checkStepDependencies(steps)` and `verifyStepPreconditions(step)`. Reads from `src/main/resources/sample-data/workflows/*.json` for deterministic offline output. | called from VALIDATE task | returns `List<StepValidation>` |
| `ExecuteTools` | function-tools class | Implements `runStep(step)` and `recordStepOutcome(stepId, outcome)`. Pure in-memory simulated execution. | called from EXECUTE task | returns `StepOutcome` / `List<StepOutcome>` |
| `NotifyTools` | function-tools class | Implements `buildRunSummary(outcomes)` and `dispatchNotification(summary)`. | called from NOTIFY task | returns `RunSummary` / `NotificationReceipt` |
| `StageGuardrail` | `before-tool-call` guardrail (registered on `PipelineOrchestrationAgent`) | Reads the in-flight task's declared stage and the current `WorkflowRunEntity` status. Rejects any tool call whose stage precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `StepCoverageScorer` | plain class (no Akka primitive) | Pure deterministic on-completion evaluator. Inputs: `RunResult`, `ExecutionResult`, `ValidationReport`. Output: `EvalResult{score, rationale}`. | called from `evalStage` | returns score |
| `WorkflowRunView` | `View` | Read model: one row per run for the UI. | `WorkflowRunEntity` events | `WorkflowRunEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record StepDefinition(String stepId, String name, List<String> dependsOn, String command) {}

record WorkflowDefinition(String workflowId, String name, List<StepDefinition> steps) {}

record StepValidation(String stepId, String status, String warning) {}

record ValidationReport(
    String workflowId,
    List<StepValidation> validations,
    boolean allPassed,
    Instant validatedAt
) {}

record StepOutcome(
    String stepId,
    String status,           // SUCCEEDED | FAILED | SKIPPED
    long durationMs,
    String detail
) {}

record ExecutionResult(
    String workflowId,
    List<StepOutcome> outcomes,
    Instant executedAt
) {}

record NotificationReceipt(String channel, String messageId, Instant sentAt) {}

record RunSummary(String workflowId, int totalSteps, int succeeded, int failed, int skipped) {}

record RunResult(
    String runId,
    String workflowId,
    RunSummary summary,
    List<StepOutcome> stepOutcomes,
    NotificationReceipt notification,
    Instant completedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record WorkflowRunRecord(
    String runId,
    Optional<String> workflowId,
    Optional<ValidationReport> validation,
    Optional<ExecutionResult> execution,
    Optional<RunResult> result,
    Optional<EvalResult> eval,
    WorkflowRunStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum WorkflowRunStatus {
    CREATED, VALIDATING, VALIDATED, EXECUTING, EXECUTED,
    NOTIFYING, NOTIFIED, EVALUATED, FAILED
}
```

Events on `WorkflowRunEntity`: `RunCreated`, `ValidateStarted`, `StepsValidated`, `ExecuteStarted`, `StepsExecuted`, `NotifyStarted`, `NotificationSent`, `RunEvaluated`, `StageGuardrailRejected`, `RunFailed`.

Every nullable lifecycle field on the `WorkflowRunRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/runs` — body `{ workflowId }` → `{ runId }`.
- `GET /api/runs` — list all runs, newest-first.
- `GET /api/runs/{id}` — one run.
- `GET /api/runs/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Workflows Process Orchestration</title>`.

The App UI tab is a two-column layout: a left rail with the live list of runs (status pill + workflow name + age) and a right pane with the selected run's detail — workflow definition, validation report table, execution outcomes, run result, eval score chip, and a guardrail-rejection log strip if any stage-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — `before-tool-call` guardrail (stage-gate)**: `StageGuardrail` is registered on `PipelineOrchestrationAgent` and runs before every tool call. It reads the in-flight `Task`'s declared stage (encoded as a constant on each function-tool class — `Stage.VALIDATE`, `Stage.EXECUTE`, `Stage.NOTIFY`) and the current `WorkflowRunEntity.status` for the run the task is bound to. The accept rule is precise: `VALIDATE` tools require `status ∈ {CREATED, VALIDATING}`; `EXECUTE` tools require `status ∈ {VALIDATED, EXECUTING}` AND `validation.isPresent()`; `NOTIFY` tools require `status ∈ {EXECUTED, NOTIFYING}` AND `execution.isPresent()`. On reject, the guardrail returns a structured `stage-violation` error to the agent loop and the workflow records a `StageGuardrailRejected{stage, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget.
- **E1 — `on-completion-eval`**: runs immediately after `NotificationSent` lands, as `evalStage` inside the workflow. `StepCoverageScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every declared step in `ValidationReport.validations` must have a matching `ExecutionResult.outcomes` entry (step coverage), every `StepOutcome.stepId` must reference a `StepDefinition.stepId` from the workflow declaration (no phantom steps), every executed step's outcome must be one of `{SUCCEEDED, FAILED, SKIPPED}` (valid outcome), and the outcome count must equal the declared step count (no silent expansion or collapse). Emits `RunEvaluated{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `PipelineOrchestrationAgent` → `prompts/pipeline-orchestration-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that stage's tool set, treat each task's typed input as the entire context for that stage, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Operator submits the seeded workflow `database-migration-v3`; within 60 s the run reaches `EVALUATED` with a `ValidationReport`, ≥ 3 step outcomes, and an eval score chip on the card.
2. **J2** — The agent's first iteration on a run calls a NOTIFY-stage tool (`dispatchNotification`) before `StepsExecuted` has been recorded (mock LLM path). `StageGuardrail` rejects the call; a `StageGuardrailRejected` event lands on the entity; the agent retries in-stage; the run eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A run whose mock-LLM trajectory produces an `ExecutionResult` containing a `stepId` absent from the workflow declaration is scored 1 with a rationale naming the phantom step; the UI flags the card.
4. **J4** — Each stage's instructions, attachments, and tool calls are visible in the per-run trace (logged at `INFO`); the VALIDATE task's log shows only VALIDATE-tool calls, the EXECUTE task's log shows only EXECUTE-tool calls, the NOTIFY task's log shows only NOTIFY-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sequential-pipeline-ops-automation-style-workflow demonstrating the
sequential-pipeline x ops-automation cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact sequential-pipeline-ops-automation-style-workflow.
Java package io.akka.samples.workflowsprocessorchestration. Akka 3.6.0. HTTP port 9807.

Components to wire (exactly):

- 1 AutonomousAgent PipelineOrchestrationAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/pipeline-orchestration-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered with
  .tools(...) — the VALIDATE, EXECUTE, and NOTIFY tool sets are ALL registered on the agent;
  stage gating is the job of StageGuardrail, NOT of conditional .tools(...) wiring. The
  before-tool-call guardrail (StageGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow WorkflowRunPipeline per runId with four stages:
  * validateStage — emits ValidateStarted on the entity, then calls componentClient
    .forAutonomousAgent(PipelineOrchestrationAgent.class, "agent-" + runId).runSingleTask(
      TaskDef.instructions("WorkflowId: " + workflowId + "\nStage: VALIDATE\nUse the
      check and verify tools to validate all declared steps.")
        .metadata("runId", runId)
        .metadata("stage", "VALIDATE")
        .taskType(WorkflowTasks.VALIDATE_STEPS)
    ). Reads forTask(taskId).result(VALIDATE_STEPS) to get ValidationReport. Writes
    WorkflowRunEntity.recordValidation(report). WorkflowSettings.stepTimeout 60s.
  * executeStage — emits ExecuteStarted, then runSingleTask with TaskDef.instructions
    (formatExecuteContext(report, workflowId)) and metadata.stage = "EXECUTE", taskType
    EXECUTE_STEPS. Writes WorkflowRunEntity.recordExecution(result). stepTimeout 60s.
  * notifyStage — emits NotifyStarted, then runSingleTask with TaskDef.instructions
    (formatNotifyContext(result, report, workflowId)) and metadata.stage = "NOTIFY", taskType
    NOTIFY_COMPLETION. Writes WorkflowRunEntity.recordResult(runResult). stepTimeout 60s.
  * evalStage — runs the deterministic StepCoverageScorer over (runResult, execution,
    validation) and writes WorkflowRunEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(WorkflowRunPipeline::error). The error step writes
  RunFailed and ends.

- 1 EventSourcedEntity WorkflowRunEntity (one per runId). State WorkflowRunRecord{runId,
  workflowId: Optional<String>, validation: Optional<ValidationReport>,
  execution: Optional<ExecutionResult>, result: Optional<RunResult>,
  eval: Optional<EvalResult>, status: WorkflowRunStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. WorkflowRunStatus enum: CREATED, VALIDATING, VALIDATED,
  EXECUTING, EXECUTED, NOTIFYING, NOTIFIED, EVALUATED, FAILED. Events:
  RunCreated{workflowId}, ValidateStarted, StepsValidated{validation},
  ExecuteStarted, StepsExecuted{execution}, NotifyStarted, NotificationSent{result},
  RunEvaluated{eval}, StageGuardrailRejected{stage, tool, reason}, RunFailed{reason}.
  Commands: create, startValidate, recordValidation, startExecute, recordExecution,
  startNotify, recordResult, recordEvaluation, recordGuardrailRejection, fail, getRun.
  emptyState() returns WorkflowRunRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View WorkflowRunView with row type WorkflowRunRow that mirrors WorkflowRunRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes WorkflowRunEntity
  events. ONE query getAllRuns: SELECT * AS runs FROM workflow_run_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * WorkflowRunEndpoint at /api with POST /runs (body {workflowId}; mints runId; calls
    WorkflowRunEntity.create(workflowId); then starts WorkflowRunPipeline with id
    "pipeline-" + runId; returns {runId}), GET /runs (list from getAllRuns, sorted
    newest-first), GET /runs/{id} (one row), GET /runs/sse (Server-Sent Events forwarded
    from the view's stream-updates), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- WorkflowTasks.java declaring three Task<R> constants:
    VALIDATE_STEPS = Task.name("Validate steps").description("Check step dependency graph
      and verify preconditions for all declared steps").resultConformsTo(ValidationReport.class);
    EXECUTE_STEPS = Task.name("Execute steps").description("Run each step in dependency
      order and record outcomes").resultConformsTo(ExecutionResult.class);
    NOTIFY_COMPLETION = Task.name("Notify completion").description("Build run summary
      and dispatch completion notification").resultConformsTo(RunResult.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Stage.java — enum {VALIDATE, EXECUTE, NOTIFY}. Each function-tool method is annotated with
  the constant stage, e.g. @FunctionTool(name = "checkStepDependencies", stage = Stage.VALIDATE)
  (use a custom annotation if the SDK's @FunctionTool does not carry a stage field — the
  guardrail reads it from a parallel registry built at startup if so).

- ValidateTools.java — @FunctionTool checkStepDependencies(List<StepDefinition>) ->
  List<StepValidation> reading dependency graph from the workflow definition; @FunctionTool
  verifyStepPreconditions(StepDefinition) -> StepValidation running precondition checks
  against in-process capability registry.

- ExecuteTools.java — @FunctionTool runStep(StepDefinition step) -> StepOutcome (simulated
  execution; reads outcome from src/main/resources/sample-data/workflows/*.json keyed by
  workflowId+stepId; records durationMs and status); @FunctionTool recordStepOutcome(
  String stepId, StepOutcome outcome) -> void persisting to in-memory log.

- NotifyTools.java — @FunctionTool buildRunSummary(List<StepOutcome>) -> RunSummary
  (aggregates totals); @FunctionTool dispatchNotification(RunSummary) -> NotificationReceipt
  (in-process stub; mints a messageId).

- StageGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.stage attribute, looks up the WorkflowRunEntity status by runId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("stage-violation: <tool> requires <precondition>, saw
  <status>"). On reject ALSO calls WorkflowRunEntity.recordGuardrailRejection(stage, tool,
  reason) so the rejection is visible in the UI's rejection-log strip and in the audit log.

- StepCoverageScorer.java — pure deterministic logic (no LLM). Inputs: RunResult,
  ExecutionResult, ValidationReport. Outputs: EvalResult with score and rationale. Four
  checks, one point per check satisfied, starting from a base of 1: step coverage (every
  StepValidation has a matching StepOutcome), phantom-step check (every StepOutcome.stepId
  appears in ValidationReport.validations[].stepId), valid-outcome check (every
  StepOutcome.status is one of SUCCEEDED/FAILED/SKIPPED), and count parity
  (outcomes.size() == validations.size()). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9807 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/workflows.jsonl with 5 seeded workflow definition lines
  covering database-migration-v3, service-deployment-canary, nightly-etl-refresh, and two extras.

- src/main/resources/sample-data/workflows/*.json — three files keyed by seeded workflow id,
  each carrying per-step simulation data with deterministic outcomes so ExecuteTools.runStep
  returns the same result across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — ops-automation
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (workflow definitions
  are operational, not person-level), decisions.authority_level = recommend-only (the run result
  is informational; a human reviews before re-running), oversight.human_in_loop = true (an
  operator reviews run outcomes before triggering follow-on actions),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "phantom-step-execution", "missing-step-coverage",
  "stage-violation", "out-of-order-notification"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/pipeline-orchestration-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Workflows Process Orchestration",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of run cards; right = selected-run detail with workflow header, validation table,
  execution outcome list, run result, eval-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Workflows Process Orchestration</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with a per-task dispatch on the Task<R> id. Each branch
  reads src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(runId)), and deserialises into the task's typed return. Each entry carries
  a "tool_calls" array the mock replays in order before returning the final typed result.
- Per-task mock-response shapes:
    validate-steps.json — 6 ValidationReport entries, each with 3-5 StepValidation items.
      Plus 1 deliberately PHASE-VIOLATING entry whose tool_calls starts with
      dispatchNotification (a NOTIFY-stage tool called during VALIDATE) — the guardrail
      rejects it; mock then falls through to a normal validate sequence. Violating entry
      selected on the first iteration of every 3rd run (modulo seed) so J2 is reproducible.
    execute-steps.json — 6 ExecutionResult entries paired with validate entries, each with
      3-5 StepOutcome items. tool_calls containing runStep + recordStepOutcome in order.
    notify-completion.json — 6 RunResult entries paired with execute entries. tool_calls
      containing buildRunSummary + dispatchNotification. Plus 1 PHANTOM-STEP entry whose
      ExecutionResult contains a stepId absent from the paired ValidationReport — evalStage
      scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(runId) helper makes per-run selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PipelineOrchestrationAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. WorkflowTasks.java MUST exist.
- Lesson 4: every workflow stage that calls an agent has an explicit stepTimeout (validateStage
  60s, executeStage 60s, notifyStage 60s, evalStage 5s, error 5s).
- Lesson 6: every nullable lifecycle field on WorkflowRunRecord is Optional<T>.
- Lesson 7: WorkflowTasks.java with VALIDATE_STEPS, EXECUTE_STEPS, NOTIFY_COMPLETION is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9807 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND themeVariables
  block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList index.
  DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (PipelineOrchestrationAgent). The
  on-completion eval is rule-based (StepCoverageScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each stage's tool set is registered on the agent, but
  StageGuardrail is the runtime mechanism that enforces the stage order.
- Stage dependency is carried by typed task results: validateStage writes ValidationReport
  onto the entity, executeStage reads it and builds the EXECUTE task's instruction context,
  notifyStage reads both. The agent itself is stateless across stages.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
