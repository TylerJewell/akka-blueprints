# SPEC — self-discover-modules

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Discover Agent.
**One-line pitch:** Submit a task; a Structurer selects and adapts a set of reasoning modules into an ordered structure, an eval hook scores it, and a Solver executes each module step to produce a grounded answer.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with two dedicated agents: one that plans (selects, adapts, composes) and one that executes (runs each module step). The Structurer owns the `ReasoningStructure` — an ordered list of `ModuleStep` records, each with a selected module name, an adapted description tailored to the task, and a role tag. The Solver works through those steps in sequence, accumulating `StepExecution` records that feed the final answer.

A single governance mechanism sits between planning and execution:

- an **on-decision eval** that scores the `ReasoningStructure` before the Solver ever runs, blocking execution when quality falls below the configured threshold and asking the Structurer to revise.

The blueprint also demonstrates a **deployer runtime-monitoring** surface: an operator can halt new task dispatches from the UI without killing a Solver that is mid-step.

## 3. User-facing flows

The user opens the App UI tab and submits a task via the form.

1. The system creates a `Task` record in `STRUCTURING` and starts a `TaskWorkflow`.
2. The `StructurerAgent` produces a `ReasoningStructure { selectedModules, adaptedSteps, compositionRationale }` and emits `StructureComposed`.
3. The **on-decision eval** scores the structure (module coverage, step coherence, completeness). If the score falls below the threshold, `StructureRejected` is emitted and the workflow asks the Structurer to revise. The Structurer may revise at most twice; a third rejection emits `TaskFailed`.
4. Once the structure passes eval, the workflow emits `StructureApproved` and the task moves to `SOLVING`.
5. The `SolverAgent` executes each module step in order, emitting `StepExecuted { stepIndex, moduleName, observation, ok }` for each. The workflow records steps as `StepExecution` entries on `TaskEntity`.
6. After all steps complete, the `SolverAgent` produces a `TaskAnswer { summary, observations, producedAt }` and emits `TaskCompleted`.
7. The operator can press **Halt new dispatches** at any time. The workflow finishes the in-flight step, then emits `TaskHaltedOperator` and moves the task to `HALTED`.

A `RequestSimulator` (TimedAction) drips a sample task every 90 seconds.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StructurerAgent` | `AutonomousAgent` | Selects, adapts, and composes reasoning modules into a `ReasoningStructure`. Revises on eval rejection. | `TaskWorkflow` | returns `ReasoningStructure` to workflow |
| `SolverAgent` | `AutonomousAgent` | Executes each `ModuleStep` in sequence and produces a `TaskAnswer`. | `TaskWorkflow` | returns typed results to workflow |
| `TaskWorkflow` | `Workflow` | Drives the structure → eval → [revise loop] → solve → record loop, plus halt and fail branches. | `TaskEndpoint`, `TaskRequestConsumer` | `TaskEntity` |
| `TaskEntity` | `EventSourcedEntity` | Holds the task lifecycle, composed structure, step executions, and final answer. | `TaskWorkflow` | `TaskView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `TaskEndpoint` (operator action) | `TaskWorkflow` (polls) |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted tasks. | `TaskEndpoint`, `RequestSimulator` | `TaskRequestConsumer` |
| `TaskView` | `View` | List-of-tasks read model for the UI. | `TaskEntity` events | `TaskEndpoint` |
| `TaskRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `TaskWorkflow` per submission. | `RequestQueue` events | `TaskWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/task-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckTaskMonitor` | `TimedAction` | Every 30 s, marks any task stuck in `STRUCTURING` or `SOLVING` past 5 minutes as `STUCK`. | scheduler | `TaskEntity` |
| `TaskEndpoint` | `HttpEndpoint` | `/api/tasks/*` — submit, get, list, SSE, operator halt/resume. | — | `TaskView`, `RequestQueue`, `TaskEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record TaskRequest(String prompt, String requestedBy) {}

record ModuleStep(
    String moduleName,
    String adaptedDescription,
    ModuleRole role,
    int stepIndex
) {}

record ReasoningStructure(
    List<String> selectedModules,
    List<ModuleStep> adaptedSteps,
    String compositionRationale,
    Optional<Integer> revisionNumber
) {}

record EvalResult(
    boolean passed,
    double score,
    Optional<String> rejectionReason
) {}

record StepExecution(
    int stepIndex,
    String moduleName,
    String observation,
    boolean ok,
    Optional<String> errorReason,
    Instant executedAt
) {}

record TaskAnswer(
    String summary,
    List<String> observations,
    Instant producedAt
) {}

record Task(
    String taskId,
    String prompt,
    TaskStatus status,
    Optional<ReasoningStructure> structure,
    Optional<EvalResult> lastEval,
    List<StepExecution> stepExecutions,
    Optional<TaskAnswer> answer,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ModuleRole { DECOMPOSE, ANALYZE, VERIFY, SYNTHESIZE, REFLECT }
enum TaskStatus { STRUCTURING, SOLVING, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`TaskEntity`)

`TaskCreated`, `StructureComposed`, `StructureRejected`, `StructureApproved`, `StructureRevised`, `StepExecuted`, `TaskCompleted`, `TaskFailed`, `TaskHaltedOperator`, `TaskFailedTimeout`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RequestQueue`)

`TaskSubmitted { taskId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/tasks` — body `{ prompt, requestedBy? }` → `202 { taskId }`. Starts a workflow.
- `GET /api/tasks` — list all tasks. Optional `?status=...`.
- `GET /api/tasks/{id}` — one task (full structure + step executions + answer).
- `GET /api/tasks/sse` — server-sent events stream of every task change.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Self-Discover <span class=\"accent\">Agent</span>"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task, operator halt/resume control, live list of tasks with status pills, expand-row to see the composed structure, per-step executions, and the final answer.

Browser title: `<title>Akka Sample: Self-Discover Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision eval** (`eval-event`, flavor `on-decision-eval`): after the Structurer produces a `ReasoningStructure`, the workflow runs a deterministic evaluator (`StructureEvaluator`) that scores the structure on three dimensions — module coverage (at least one DECOMPOSE and one SYNTHESIZE step), step coherence (no duplicate module names unless role differs), and completeness (2–8 steps). Score 0.0–1.0; threshold 0.7. Below threshold: `StructureRejected` emitted, loop back to `structureStep` with the `rejectionReason` appended to the Structurer's context. Non-blocking to the user (the system revises automatically); blocking to the Solver (the Solver never runs until the structure passes).
- **HO1 — deployer runtime monitoring** (`hotl`, flavor `deployer-runtime-monitoring`): an operator dashboard pane shows every in-flight task, its current step index, and the last step's observation. Two buttons — Halt new dispatches, Resume — drive `SystemControlEntity`. Every `TaskWorkflow` reads the flag in its `checkHaltStep` at the top of each solve iteration. On `halted=true`, the workflow finishes the in-flight step, then emits `TaskHaltedOperator` and ends the task in `HALTED`.

## 9. Agent prompts

- `StructurerAgent` → `prompts/structurer.md`. Selects, adapts, composes, and revises.
- `SolverAgent` → `prompts/solver.md`. Executes module steps.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Explain why gradient descent can converge to a local minimum rather than a global minimum." Task progresses `STRUCTURING → SOLVING → COMPLETED` within ~3 minutes. Expanded view shows a `ReasoningStructure` with 3–6 steps, at least one DECOMPOSE and one SYNTHESIZE step, an `EvalResult` with `passed=true`, and a `TaskAnswer` with a non-empty summary and 3–5 observation bullets.
2. **J2** — Structurer initially produces a structure the eval rejects (score < 0.7). The `StructureRejected` event appears in the task; the Structurer revises and the second structure passes. Task eventually completes.
3. **J3** — Submit a task and click **Halt new dispatches** while it is `SOLVING`. The in-flight module step finishes; no further steps execute; the task ends in `HALTED`.
4. **J4** — Structurer exhausts two revisions and a third structure still fails eval. Task ends in `FAILED` with `failureReason = "structure eval failed after maximum revisions"`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-discover-modules demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-self-discover-modules.
Java package io.akka.samples.selfdiscoveragent. Akka 3.6.0. HTTP port 9914.

Components to wire (exactly):
- 2 AutonomousAgents:
  * StructurerAgent — definition() with capabilities:
      capability(TaskAcceptance.of(SELECT_MODULES).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_STRUCTURE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(REVISE_STRUCTURE).maxIterationsPerTask(2)).
    System prompt from prompts/structurer.md.
    SELECT_MODULES returns List<String> (module names).
    COMPOSE_STRUCTURE returns ReasoningStructure.
    REVISE_STRUCTURE returns ReasoningStructure (revised, increments revisionNumber).
  * SolverAgent — capabilities:
      capability(TaskAcceptance.of(EXECUTE_STEP).maxIterationsPerTask(2)) and
      capability(TaskAcceptance.of(COMPOSE_ANSWER).maxIterationsPerTask(2)).
    Prompt from prompts/solver.md.
    EXECUTE_STEP returns StepExecution.
    COMPOSE_ANSWER returns TaskAnswer.

- 1 Workflow TaskWorkflow with steps:
  selectStep -> composeStep -> evalStep ->
  [if rejected and revisions < 2: reviseStep -> evalStep]
  [if rejected and revisions == 2: failStep]
  [if passed: checkHaltStep -> executeStepN -> recordStepStep -> decideStep
   -> [loop back to checkHaltStep or composeAnswerStep / failStep / haltedStep]]
  Step timeouts (override settings() per Lesson 4):
    selectStep ofSeconds(45), composeStep ofSeconds(60),
    reviseStep ofSeconds(60), executeStepN ofSeconds(90),
    composeAnswerStep ofSeconds(60). defaultStepRecovery(maxRetries(2).failoverTo
    (TaskWorkflow::error)).
  evalStep runs StructureEvaluator.evaluate(structure); on passed=false records
  StructureRejected via TaskEntity.rejectStructure(reason, score), increments
  revisionCount, and loops to reviseStep if revisionCount < 2, else transitions
  to failStep.
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits TaskHaltedOperator on TaskEntity).
  executeStepN iterates through structure.adaptedSteps in order. For each step,
  calls SolverAgent with EXECUTE_STEP(moduleName, adaptedDescription, accumulated
  observations so far), then calls TaskEntity.recordStep(execution).
  composeAnswerStep calls SolverAgent with COMPOSE_ANSWER(stepExecutions).
  haltedStep emits TaskHaltedOperator; failStep emits TaskFailed.

- 1 EventSourcedEntity TaskEntity holding Task state. emptyState() returns
  Task.initial with empty stepExecutions list and no Optional fields populated.
  Commands: createTask, composeStructure, rejectStructure, approveStructure,
  reviseStructure, recordStep, completeTask, failTask, haltOperator,
  timeoutFail, getTask. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RequestQueue with command enqueueTask(taskId, prompt,
  requestedBy) emitting TaskSubmitted.

- 1 View TaskView with row type TaskRow (mirror of Task minus heavy step payloads —
  truncate to last 3 step executions plus counts; the UI fetches the full task
  by id on click). Table updater consumes TaskEntity events.
  ONE query getAllTasks SELECT * AS tasks FROM task_view. No WHERE status
  filter — caller filters client-side (Lesson 2).

- 1 Consumer TaskRequestConsumer subscribed to RequestQueue events; on
  TaskSubmitted starts a TaskWorkflow with taskId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/task-prompts.jsonl and calls
    RequestQueue.enqueueTask.
  * StuckTaskMonitor — every 30s, queries TaskView.getAllTasks, filters
    STRUCTURING or SOLVING tasks whose createdAt is older than 5 minutes,
    calls TaskEntity.timeoutFail; TaskWorkflow polls TaskEntity.getTask in
    its decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * TaskEndpoint at /api with POST /tasks, GET /tasks (filters client-side
    from getAllTasks), GET /tasks/{id}, GET /tasks/sse,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- StructurerTasks.java declaring three Task<R> constants: SELECT_MODULES
  (resultConformsTo List<String>), COMPOSE_STRUCTURE (ReasoningStructure),
  REVISE_STRUCTURE (ReasoningStructure).
- SolverTasks.java declaring two Task<R> constants: EXECUTE_STEP
  (resultConformsTo StepExecution), COMPOSE_ANSWER (TaskAnswer).
- Domain records as listed in SPEC §5, plus ModuleRole enum and TaskStatus enum.
- application/StructureEvaluator.java — deterministic scorer.
  Rules: score starts at 1.0. Deduct 0.3 if no step has role DECOMPOSE.
  Deduct 0.3 if no step has role SYNTHESIZE. Deduct 0.2 if fewer than 2
  steps. Deduct 0.2 if more than 8 steps. Deduct 0.1 for each duplicate
  moduleName where role also matches (max 0.2 deduction). Threshold 0.7.
  Returns EvalResult{passed, score, Optional<String> rejectionReason}.
  rejectionReason is a comma-separated list of the triggered rules.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9914 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-prompts.jsonl with 8 canned task
  prompts covering reasoning, analysis, explanation, and problem-solving tasks.
- src/main/resources/sample-data/module-library.jsonl — 12 named reasoning
  modules each with a default description and allowed roles. Modules include:
  critical-analysis, decomposition, analogy-mapping, evidence-weighing,
  hypothesis-generation, assumption-checking, step-by-step-planning,
  causal-reasoning, perspective-taking, synthesis, self-reflection,
  verification.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1, HO1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance.capabilities;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/structurer.md, prompts/solver.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Self-Discover Agent",
  one-line pitch, prerequisites (integration form host-software requirement:
  None), generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-
  mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for structure, step executions, and
  answer). Browser title exactly:
  <title>Akka Sample: Self-Discover Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        SELECT_MODULES → 4–6 module name lists.
        COMPOSE_STRUCTURE → 4–6 ReasoningStructure entries spanning 3–6
        steps, mixing roles including DECOMPOSE and SYNTHESIZE.
        REVISE_STRUCTURE → 3–4 ReasoningStructure entries that add or
        swap a module to fix the typical eval rejection.
        EXECUTE_STEP → 6 StepExecution entries, one per module type,
        ok=true, 3–5 sentence observations.
        COMPOSE_ANSWER → 4 TaskAnswer entries with 60–120 word summaries
        and 3–5 observation bullets.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (selectStep, composeStep, reviseStep,
  executeStepN, composeAnswerStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the Task entity state (structure, lastEval, answer, failureReason,
  haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion StructurerTasks.java and
  SolverTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9914 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the
  box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
