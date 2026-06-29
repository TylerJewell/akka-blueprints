# SPEC — self-discover-planner

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Self-Discover Workflow.
**One-line pitch:** Submit a task; a Planner selects and sequences relevant reasoning modules from a fixed library, a PlanEvaluator scores the resulting plan before execution begins, an Executor runs each module step in order, and a Synthesiser assembles the final answer.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern in its clearest form: the planning phase produces an explicit, inspectable artefact (a `ReasoningPlan`) before any execution begins. The plan is not an internal prompt fragment — it is a first-class domain record persisted on `SolveEntity` and evaluated by a dedicated agent before the Executor ever runs.

The blueprint demonstrates one governance mechanism wired into that boundary:

- an **on-decision eval-event** that scores the `ReasoningPlan` against a quality rubric (module coverage, step coherence, absence of circular dependencies) before the workflow enters the execute phase. A plan below threshold is returned to the `PlannerAgent` for revision; only a passing plan advances to execution.

The module library is fixed at startup (seeded from `sample-data/modules.jsonl`) and held in `ModuleRegistry`. Modules have a kind (`DECOMPOSE`, `ANALYSE`, `COMPARE`, `GENERATE`, `VERIFY`, `REFLECT`) and a short description. The `PlannerAgent` selects a subset (3–7 modules) and arranges them into an ordered `ReasoningPlan`.

## 3. User-facing flows

The user opens the App UI tab and submits a task via the form.

1. The system creates a `Solve` record in `PLANNING` and starts a `SolveWorkflow`.
2. The `PlannerAgent` reads the task and the module registry, then produces a `ReasoningPlan { selectedModules, stepOrder, rationale }` and emits `SolvePlanned`.
3. The `SolveWorkflow` enters the evaluation gate. The `PlanEvaluatorAgent` scores the plan against the quality rubric and returns a `PlanEval { score, verdict, feedback }`.
   - If `verdict = PASS`, the workflow emits `PlanAccepted` and advances to execution.
   - If `verdict = FAIL`, the workflow emits `PlanRejected { feedback }` and loops back to the `PlannerAgent` with the evaluator's feedback. After two failed revisions the workflow emits `SolveFailed`.
4. The workflow enters the executor loop. For each step in `stepOrder`:
   - The `ExecutorAgent` runs the module and returns a `ModuleResult { moduleId, kind, output }`.
   - The workflow emits `ModuleExecuted { stepIndex, moduleId, result }` on `SolveEntity`.
5. Once all steps complete, the `SynthesiserAgent` reads the full `ExecutionLog` and produces a `TaskAnswer { summary, evidence }`. The workflow emits `SolveCompleted`.
6. At any point a `StuckSolveMonitor` can mark the solve `STUCK` if no progress is made within 5 minutes; the workflow's check step exits with `SolveFailedTimeout`.

A `RequestSimulator` (TimedAction) drips a sample task every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Selects and sequences reasoning modules into a `ReasoningPlan`. Accepts revision feedback from the evaluator. | `SolveWorkflow` | returns `ReasoningPlan` to workflow |
| `PlanEvaluatorAgent` | `AutonomousAgent` | Scores a `ReasoningPlan` against the quality rubric. Returns `PlanEval`. | `SolveWorkflow` | returns `PlanEval` to workflow |
| `ExecutorAgent` | `AutonomousAgent` | Executes a single module step given the module spec and the running execution log. Returns `ModuleResult`. | `SolveWorkflow` | returns `ModuleResult` to workflow |
| `SynthesiserAgent` | `AutonomousAgent` | Combines all `ModuleResult` entries in the `ExecutionLog` into a `TaskAnswer`. | `SolveWorkflow` | returns `TaskAnswer` to workflow |
| `SolveWorkflow` | `Workflow` | Drives: compose-plan → evaluate-plan → [reject loop] → execute-modules (per-step loop) → synthesise → complete. | `SolveEndpoint`, `SolveRequestConsumer` | `SolveEntity` |
| `SolveEntity` | `EventSourcedEntity` | Holds the solve lifecycle, the accepted `ReasoningPlan`, the `ExecutionLog`, and the final `TaskAnswer`. | `SolveWorkflow` | `SolveView` |
| `ModuleRegistry` | `EventSourcedEntity` | Holds the canonical list of `ReasoningModule` entries. Seeded at startup. Single instance keyed by `"global"`. | startup seeder, `SolveWorkflow` | `PlannerAgent` context |
| `RequestQueue` | `EventSourcedEntity` | Audit log of submitted solve requests. | `SolveEndpoint`, `RequestSimulator` | `SolveRequestConsumer` |
| `SolveView` | `View` | List-of-solves read model for the UI. | `SolveEntity` events | `SolveEndpoint` |
| `SolveRequestConsumer` | `Consumer` | Subscribes to `RequestQueue` events; starts a `SolveWorkflow` per submission. | `RequestQueue` events | `SolveWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads next line from `sample-events/task-prompts.jsonl` and enqueues it. | scheduler | `RequestQueue` |
| `StuckSolveMonitor` | `TimedAction` | Every 30 s, marks solves stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `SolveEntity` |
| `SolveEndpoint` | `HttpEndpoint` | `/api/solves/*` — submit, get, list, SSE. | — | `SolveView`, `RequestQueue`, `SolveEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record SolveRequest(String prompt, String requestedBy) {}

record ReasoningModule(
    String moduleId,
    ModuleKind kind,
    String name,
    String description
) {}

record ReasoningPlan(
    List<String> selectedModuleIds,
    List<PlanStep> stepOrder,
    String rationale,
    Optional<String> revisionNote
) {}

record PlanStep(
    int index,
    String moduleId,
    String objective,
    List<String> inputsFrom
) {}

record PlanEval(
    double score,
    PlanVerdict verdict,
    String feedback
) {}

record ModuleResult(
    String moduleId,
    ModuleKind kind,
    String output,
    boolean ok,
    Optional<String> errorReason
) {}

record ExecutionLog(List<ExecutionStep> steps) {}

record ExecutionStep(
    int stepIndex,
    String moduleId,
    ModuleKind kind,
    String objective,
    ModuleResult result,
    Instant executedAt
) {}

record TaskAnswer(String summary, List<String> evidence, Instant producedAt) {}

record Solve(
    String solveId,
    String prompt,
    SolveStatus status,
    Optional<ReasoningPlan> plan,
    Optional<PlanEval> planEval,
    Optional<ExecutionLog> executionLog,
    Optional<TaskAnswer> answer,
    Optional<String> failureReason,
    int planRevisionCount,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ModuleKind { DECOMPOSE, ANALYSE, COMPARE, GENERATE, VERIFY, REFLECT }
enum PlanVerdict { PASS, FAIL }
enum SolveStatus { PLANNING, EVALUATING, EXECUTING, COMPLETED, FAILED, STUCK }
```

### Events (`SolveEntity`)

`SolveCreated`, `SolvePlanned`, `PlanAccepted`, `PlanRejected`, `ModuleExecuted`, `SolveCompleted`, `SolveFailed`, `SolveFailedTimeout`.

### Events (`ModuleRegistry`)

`RegistrySeeded { modules: List<ReasoningModule> }`.

### Events (`RequestQueue`)

`SolveSubmitted { solveId, prompt, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/solves` — body `{ prompt, requestedBy? }` → `202 { solveId }`. Starts a workflow.
- `GET /api/solves` — list all solves. Optional `?status=...`.
- `GET /api/solves/{id}` — one solve (full plan + execution log + answer).
- `GET /api/solves/sse` — server-sent events stream of every solve change.
- `GET /api/registry` — `{ modules: List<ReasoningModule> }`. Returns the full module list from `ModuleRegistry`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Self-Discover Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a task, live list of solves with status pills, expand-row to see the reasoning plan, module execution log, plan-eval score, and the final answer.

Browser title: `<title>Akka Sample: Self-Discover Workflow</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision plan-quality eval** (`eval-event`, flavor `on-decision-eval`): the `SolveWorkflow` calls `PlanEvaluatorAgent` after every `SolvePlanned` event and before any `ModuleExecuted` event. The evaluator scores the plan on three dimensions — module coverage (does the selected set cover the task?), step coherence (are inputs and outputs of adjacent steps compatible?), dependency validity (no circular `inputsFrom` references). A plan scoring below 0.65 (on a 0–1 scale) receives `PlanVerdict.FAIL` and the workflow returns it to the `PlannerAgent` with the evaluator's `feedback`. Only a `PlanVerdict.PASS` plan causes `PlanAccepted` to be emitted and execution to begin. Revision budget: 2 rejections; a third triggers `SolveFailed`.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Selects modules; arranges steps; revises on feedback.
- `PlanEvaluatorAgent` → `prompts/plan-evaluator.md`. Scores the plan; returns structured verdict.
- `ExecutorAgent` → `prompts/executor.md`. Runs one module step; returns typed result.
- `SynthesiserAgent` → `prompts/synthesiser.md`. Combines module results into a final answer.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "What are the tradeoffs between consistency and availability in distributed databases?" Task progresses `PLANNING → EVALUATING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a `ReasoningPlan` with 3–7 steps, a `PlanEval` with `verdict = PASS`, an `ExecutionLog` with one entry per plan step, and a non-empty `TaskAnswer`.
2. **J2** — Submit a task for which the first plan draft scores below threshold. The `PlanRejected` event is visible on the expanded solve. The planner produces a revised plan; after the revised plan passes evaluation, execution proceeds normally.
3. **J3** — Submit a task and observe via the SSE stream that `SolvePlanned` fires before any `ModuleExecuted` event, and `PlanAccepted` fires before any `ModuleExecuted` event — proving the eval gate is honoured in the event sequence.
4. **J4** — Submit a task whose module execution produces output containing an `AKIA...` key shape. The `ExecutionStep.result.output` in the stored log shows `[REDACTED:aws-access-key]`; the raw key is absent from `TaskAnswer.summary` and from all SSE payloads.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-discover-planner demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-self-discover-planner.
Java package io.akka.samples.selfdiscoverworkflow. Akka 3.6.0. HTTP port 9699.

Components to wire (exactly):
- 4 AutonomousAgents:
  * PlannerAgent — definition() with two capabilities:
      capability(TaskAcceptance.of(COMPOSE_PLAN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(REVISE_PLAN).maxIterationsPerTask(3)).
    System prompt from prompts/planner.md. COMPOSE_PLAN returns ReasoningPlan.
    REVISE_PLAN returns ReasoningPlan (revised, revisionNote populated).
  * PlanEvaluatorAgent — capability(TaskAcceptance.of(EVALUATE_PLAN).maxIterationsPerTask(2)).
    Prompt from prompts/plan-evaluator.md. Returns PlanEval.
  * ExecutorAgent — capability(TaskAcceptance.of(EXECUTE_MODULE).maxIterationsPerTask(2)).
    Prompt from prompts/executor.md. Returns ModuleResult.
  * SynthesiserAgent — capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(2)).
    Prompt from prompts/synthesiser.md. Returns TaskAnswer.

- 1 Workflow SolveWorkflow with steps:
  composePlanStep -> evaluatePlanStep -> [reject loop: revisePlanStep ->
  evaluatePlanStep] -> [execute loop: executeModuleStep (one per plan step)] ->
  synthesiseStep -> completeStep | failStep | stuckStep.
  Step timeouts (override settings() per Lesson 4):
    composePlanStep ofSeconds(60), revisePlanStep ofSeconds(60),
    evaluatePlanStep ofSeconds(45), executeModuleStep ofSeconds(90)
    (covers any module call), synthesiseStep ofSeconds(60),
    completeStep ofSeconds(30). defaultStepRecovery(maxRetries(2).failoverTo
    (SolveWorkflow::error)).
  composePlanStep calls PlannerAgent COMPOSE_PLAN, emits SolvePlanned.
  evaluatePlanStep calls PlanEvaluatorAgent EVALUATE_PLAN; on PASS emits
    PlanAccepted and advances to execute loop; on FAIL emits PlanRejected with
    feedback and routes to revisePlanStep if revisionCount < 2, else failStep.
  revisePlanStep calls PlannerAgent REVISE_PLAN (passes the evaluator feedback),
    emits SolvePlanned (revision), increments planRevisionCount.
  executeModuleStep iterates over stepOrder; for each step calls ExecutorAgent
    EXECUTE_MODULE (passes module spec + ExecutionLog so far), emits
    ModuleExecuted.
  synthesiseStep calls SynthesiserAgent SYNTHESISE (passes full ExecutionLog),
    emits SolveCompleted.
  stuckStep: workflow reads SolveEntity status; if STUCK, emits SolveFailedTimeout
    and ends.

- 1 EventSourcedEntity SolveEntity holding Solve state. emptyState() returns
  Solve.initial("", null). Commands: createSolve, recordPlan, acceptPlan,
  rejectPlan, recordModuleExecution, completeSolve, failSolve, timeoutFail,
  getSolve. Events as listed in SPEC §5.

- 1 EventSourcedEntity ModuleRegistry keyed by literal "global". State
  ModuleRegistryState { List<ReasoningModule> modules }. Commands: seedRegistry,
  getModules. Events: RegistrySeeded.
  Seeded at startup by Bootstrap.java reading sample-data/modules.jsonl.

- 1 EventSourcedEntity RequestQueue with command enqueueTask(solveId, prompt,
  requestedBy) emitting SolveSubmitted.

- 1 View SolveView with row type SolveRow (mirror of Solve minus heavy
  executionLog — truncate to last 3 ExecutionStep entries plus counts; the UI
  fetches the full solve by id on click). Table updater consumes SolveEntity
  events. ONE query getAllSolves SELECT * AS solves FROM solve_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer SolveRequestConsumer subscribed to RequestQueue events; on
  SolveSubmitted starts a SolveWorkflow with solveId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/task-prompts.jsonl and calls
    RequestQueue.enqueueTask.
  * StuckSolveMonitor — every 30s, queries SolveView.getAllSolves, filters
    EXECUTING solves whose createdAt is older than 5 minutes, calls
    SolveEntity.timeoutFail; SolveWorkflow's stuckStep exits when it reads
    status == STUCK.

- 2 HttpEndpoints:
  * SolveEndpoint at /api with POST /solves, GET /solves (filters client-side
    from getAllSolves), GET /solves/{id}, GET /solves/sse, GET /registry (returns
    ModuleRegistry.getModules), and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring two Task<R> constants: COMPOSE_PLAN
  (resultConformsTo ReasoningPlan), REVISE_PLAN (resultConformsTo ReasoningPlan).
- EvaluatorTasks.java declaring one Task<R>: EVALUATE_PLAN
  (resultConformsTo PlanEval).
- ExecutorTasks.java declaring one Task<R>: EXECUTE_MODULE
  (resultConformsTo ModuleResult).
- SynthesiserTasks.java declaring one Task<R>: SYNTHESISE
  (resultConformsTo TaskAnswer).
- Domain records as listed in SPEC §5.
- application/OutputScrubber.java — deterministic regex/entropy scrubber applied
  to ModuleResult.output before ModuleExecuted is emitted.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36},
  JWT ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+,
  sk-[A-Za-z0-9]{32,}, Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback
  for tokens ≥ 32 chars whose Shannon entropy > 4.5 bits/char.
  Replacements: [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9699 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/task-prompts.jsonl with 8 canned task
  prompts covering analysis, comparison, and generative reasoning tasks.
- src/main/resources/sample-data/modules.jsonl — 12 reasoning module entries
  spanning all six ModuleKind values (DECOMPOSE, ANALYSE, COMPARE, GENERATE,
  VERIFY, REFLECT) with names and descriptions.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at project root with 1 control (E1) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at project root, pre-filling purpose, data, decisions,
  failure, oversight, operations, and compliance.capabilities; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/plan-evaluator.md, prompts/executor.md,
  prompts/synthesiser.md loaded at agent startup as system prompts.
- README.md at project root: title "Akka Sample: Self-Discover Workflow",
  one-line pitch, prerequisites (integration form host-software: None),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + live solve list with status pills; expand-on-click
  shows the reasoning plan, per-step execution log, plan-eval score, and
  the final answer). Browser title exactly:
  <title>Akka Sample: Self-Discover Workflow</title>. No subtitle on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI (1password://, aws-secretsmanager://, vault://).
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java with per-agent dispatch on agent class name
  and Task<R> id. Each branch reads from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    planner.json — two lists keyed by task id:
      "COMPOSE_PLAN" → 4 ReasoningPlan entries with 3–7 steps each spanning
        multiple ModuleKind values.
      "REVISE_PLAN" → 3 ReasoningPlan entries where revisionNote is populated
        and the module selection is meaningfully different.
    plan-evaluator.json — 5 PlanEval entries; 3 with verdict PASS (scores
      0.70–0.90) and 2 with verdict FAIL (scores 0.40–0.60, feedback citing
      a concrete deficiency such as "missing VERIFY step" or "circular
      inputsFrom reference").
    executor.json — 8 ModuleResult entries, ok=true, one entry per ModuleKind.
      ONE entry's output must include the literal substring "AKIAIOSFODNN7EXAMPLE"
      so the J4 scrubber test fires.
    synthesiser.json — 4 TaskAnswer entries with 60–120 word summaries and
      3–5 evidence bullets citing module ids.
- MockModelProvider.seedFor(solveId) makes selection deterministic per solve id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every step that
  calls an agent.
- Lesson 6: Optional<T> for every nullable field on a View row record and on
  the Solve entity state.
- Lesson 7: AutonomousAgent requires companion task-constant files
  (PlannerTasks, EvaluatorTasks, ExecutorTasks, SynthesiserTasks).
- Lesson 8: model-name values verified against provider lineup.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9699 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND
  themeVariables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow; no key value
  written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. No zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
