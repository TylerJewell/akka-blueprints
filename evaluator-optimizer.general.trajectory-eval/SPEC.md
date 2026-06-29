# SPEC — trajectory-eval

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Trajectory Evaluation.
**One-line pitch:** Submit a task scenario; a runner agent executes it by calling tools in sequence; an evaluator agent compares the resulting trajectory against a stored reference path and returns a pass/fail verdict with per-step deviation details.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to agent-behavior regression testing: a Workflow alternates between a generator agent (`RunnerAgent`) and a reviewer agent (`EvaluatorAgent`), feeding each deviation report back into the next run until the trajectory converges to the reference or the retry ceiling is reached. The blueprint also demonstrates three governance mechanisms — an **eval-periodic** monitor that records each cycle's verdict for downstream regression tracking, an **output guardrail** that gates each trajectory against a deterministic step-count limit before the evaluator runs, and a **halt** that ends the loop at the retry ceiling without leaving the evaluation in a degenerate state.

## 3. User-facing flows

The user opens the App UI tab and submits a task scenario (a scenario id plus an optional task description override).

1. The system looks up the reference path for the scenario in `ReferenceStore`, creates an `Evaluation` record in `RUNNING`, and starts a `TrajectoryWorkflow`.
2. The Runner executes the task for attempt #1, producing a `RecordedTrajectory` — an ordered list of `ToolCall` records each with a tool name, inputs, and output.
3. The output guardrail checks the trajectory step count against the configured maximum. Over-limit trajectories are short-circuited back to the Runner with a structured feedback note; they never reach the Evaluator.
4. The Evaluator compares the trajectory against the reference path and returns either `PASS` with a one-line rationale, or `FAIL` with a typed `DeviationReport` (a list of `Deviation` records each naming the step index, the expected tool call, and the actual tool call).
5. On `PASS`, the workflow transitions the evaluation to `PASSED` with the successful trajectory and the evaluator's rationale.
6. On `FAIL`, the workflow records the attempt, the guardrail verdict, the deviation report, and the evaluator's verdict on the entity, then calls the Runner again with the deviation report attached. The Runner produces attempt #2 using the feedback to adjust its approach.
7. If the loop reaches `maxAttempts` (default 4) without a `PASS`, the halt mechanism activates: the workflow ends with `FAILED_FINAL`, the attempt with the fewest deviations is preserved on the entity along with every trajectory and deviation report for audit, and an `EvalRecorded` event captures the failure.

A `ScenarioSimulator` (TimedAction) drips a canned scenario every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RunnerAgent` | `AutonomousAgent` | Executes a task by calling tools in an ordered sequence; accepts prior deviation report on re-runs. | `TrajectoryWorkflow` | returns `RecordedTrajectory` to workflow |
| `EvaluatorAgent` | `AutonomousAgent` | Compares a trajectory against the reference path; returns `PASS` or `FAIL` with a typed `DeviationReport`. | `TrajectoryWorkflow` | returns `TrajectoryVerdict` to workflow |
| `TrajectoryWorkflow` | `Workflow` | Runs the execute → guardrail → evaluate → retry loop; halts at the ceiling. | `TrajectoryEndpoint`, `ScenarioConsumer` | `EvaluationEntity` |
| `EvaluationEntity` | `EventSourcedEntity` | Holds the evaluation lifecycle, every trajectory attempt, every deviation report, and the final outcome. | `TrajectoryWorkflow` | `EvaluationsView` |
| `ReferenceStore` | `EventSourcedEntity` | Holds the set of reference paths keyed by scenario id; supports add, update, and retrieval. | `TrajectoryEndpoint` | `TrajectoryWorkflow` |
| `EvaluationsView` | `View` | List-of-evaluations read model. | `EvaluationEntity` events | `TrajectoryEndpoint` |
| `ScenarioConsumer` | `Consumer` | Subscribes to scenario-submission events; starts a workflow per submission. | `ReferenceStore` events | `TrajectoryWorkflow` |
| `ScenarioSimulator` | `TimedAction` | Drips a sample scenario every 60 s from `sample-events/task-scenarios.jsonl`. | scheduler | `ReferenceStore`, `TrajectoryEndpoint` |
| `EvalSampler` | `TimedAction` | Every 30 s, queries `EvaluationsView`, records an `EvalRecorded` event for any cycle completed since the last tick. | scheduler | `EvaluationEntity` |
| `TrajectoryEndpoint` | `HttpEndpoint` | `/api/evaluations/*` — submit, get, list, SSE; plus `/api/references/*` for reference-path management; plus `/api/metadata/*`. | — | `EvaluationsView`, `ReferenceStore`, `EvaluationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record TaskScenario(String scenarioId, String taskDescription, String requestedBy) {}

record ToolCall(String toolName, Map<String, Object> inputs, String output, Instant calledAt) {}

record RecordedTrajectory(List<ToolCall> steps, int stepCount, Instant recordedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record Deviation(int stepIndex, String expectedTool, String actualTool, String note) {}

record DeviationReport(List<Deviation> deviations, String overallRationale) {}

record TrajectoryVerdict(
    EvalVerdict verdict,
    DeviationReport report,
    int deviationCount,
    Instant evaluatedAt
) {}

record TrajectoryAttempt(
    int attemptNumber,
    RecordedTrajectory trajectory,
    GuardrailVerdict guardrail,
    Optional<TrajectoryVerdict> verdict
) {}

record ReferencePath(String scenarioId, List<ToolCall> steps, Instant registeredAt, Optional<Instant> updatedAt) {}

record Evaluation(
    String evaluationId,
    String scenarioId,
    String taskDescription,
    int maxAttempts,
    EvaluationStatus status,
    List<TrajectoryAttempt> attempts,
    Optional<Integer> passedAttemptNumber,
    Optional<RecordedTrajectory> passedTrajectory,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EvaluationStatus { RUNNING, EVALUATING, PASSED, FAILED_FINAL }

enum EvalVerdict { PASS, FAIL }
```

### Events (on `EvaluationEntity`)

`EvaluationCreated`, `TrajectoryRecorded`, `TrajectoryGuardrailVerdictRecorded`, `TrajectoryEvaluated`, `EvaluationPassed`, `EvaluationFailedFinal`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/evaluations` — body `{ scenarioId, taskDescription?, requestedBy? }` → `{ evaluationId }`. Starts a workflow.
- `GET /api/evaluations` — list all evaluations. Optional `?status=RUNNING|EVALUATING|PASSED|FAILED_FINAL`.
- `GET /api/evaluations/{id}` — one evaluation (including every attempt and every verdict).
- `GET /api/evaluations/sse` — server-sent events stream of every evaluation change.
- `POST /api/references` — body `{ scenarioId, steps[] }` → `{ scenarioId }`. Registers or replaces a reference path.
- `GET /api/references/{scenarioId}` — retrieve a reference path.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Trajectory Evaluation"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, guardrail = red, halt = red).
- **App UI** — form to submit a scenario, live list of evaluations with status pills, click-to-expand per-attempt timeline showing each trajectory's steps, the guardrail verdict, the evaluator's verdict, and the deviation report.

Browser title: `<title>Akka Sample: Trajectory Evaluation</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `RunnerAgent`): a deterministic check that the trajectory's step count is at or below the configured maximum (`trajectory-eval.runner.max-steps`, default 20). Over-limit trajectories short-circuit back to the Runner with a structured feedback note (`reasonCode = OVER_STEP_LIMIT`); they never reach the Evaluator. Enforcement: blocking.
- **E1 — eval-periodic** (`performance-monitor`): every cycle's verdict is recorded as an `EvalRecorded` event with `{ attemptNumber, verdict, deviationCount, stepLimitExceeded }`. The `EvalSampler` TimedAction is the canonical writer; the workflow also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/evaluations/{id}`.
- **HT1 — halt** (`graceful-degradation`): when the loop reaches `maxAttempts` without a `PASS`, the workflow ends with `EvaluationFailedFinal`. The entity preserves every trajectory attempt, every deviation report, the attempt with the fewest deviations, and a structured failure reason. The system never discards trajectory data or terminates abruptly; the halt is observable end-to-end. Enforcement: system-level.

## 9. Agent prompts

- `RunnerAgent` → `prompts/runner.md`. Executes a task by calling tools in an ordered sequence; on a re-run call, takes the prior `DeviationReport` as input and adjusts its tool-call strategy to reduce deviations from the reference.
- `EvaluatorAgent` → `prompts/evaluator.md`. Compares a recorded trajectory against the reference path step by step; returns `PASS` with a one-line rationale or `FAIL` with a typed `DeviationReport` listing each step that diverged.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a scenario; evaluation progresses `RUNNING` → `EVALUATING` → `PASSED` within the retry ceiling; the App UI shows every attempt's trajectory and verdict.
2. **J2 — halt at ceiling** — Submit a scenario whose reference path the runner cannot match (test mode forces `FAIL` every attempt); evaluation progresses through every attempt and lands in `FAILED_FINAL` with the closest-match trajectory preserved and a structured failure reason.
3. **J3 — guardrail block** — Submit a scenario configured to produce an over-limit trajectory; the guardrail short-circuits with `reasonCode = OVER_STEP_LIMIT`, the Runner re-runs with feedback, and the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any evaluation shows one `EvalRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named trajectory-eval demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-trajectory-eval.
Java package io.akka.samples.trajectoryevaluation. Akka 3.6.0. HTTP port 9235.

Components to wire (exactly):
- 2 AutonomousAgents:
  * RunnerAgent — definition() with
    capability(TaskAcceptance.of(EXECUTE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(RE_EXECUTE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/runner.md. Returns RecordedTrajectory{steps,
    stepCount, recordedAt} for both EXECUTE and RE_EXECUTE. The RE_EXECUTE task
    takes (scenarioId, taskDescription, referencePath, priorDeviationReport) as
    inputs.
  * EvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)). System
    prompt from prompts/evaluator.md. Returns TrajectoryVerdict{verdict, report,
    deviationCount, evaluatedAt} where verdict is the EvalVerdict enum (PASS | FAIL)
    and deviationCount is a non-negative integer.

- 1 Workflow TrajectoryWorkflow with steps:
    startStep -> executeStep -> guardrailStep ->
    [guardrail FAIL? executeStep again with structured feedback : evaluateStep] ->
    [verdict PASS? passStep : (attemptCount < maxAttempts ?
       executeStep with deviation report attached : failStep)] -> END.
  executeStep calls forAutonomousAgent(RunnerAgent.class, evaluationId)
    .runSingleTask(EXECUTE or RE_EXECUTE) then forTask(taskId).result(EXECUTE or
    RE_EXECUTE). evaluateStep calls forAutonomousAgent(EvaluatorAgent.class,
    evaluationId).runSingleTask(EVALUATE). passStep emits EvaluationPassed.
    failStep emits EvaluationFailedFinal with the attempt that had the fewest
    deviations as best-of and a structured failureReason. Override settings()
    with stepTimeout(60s) on executeStep and evaluateStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    trajectory.stepCount() <= maxSteps. On FAIL, emits
    TrajectoryGuardrailVerdictRecorded with verdict.passed = false and
    reasonCode = "OVER_STEP_LIMIT", then transitions back to executeStep with
    structured feedback DeviationReport(List.of(Deviation(-1, "step-count",
    "exceeded", "Trajectory exceeded the configured step limit; reduce tool calls.")),
    "Step count exceeds configured maximum.").

- 1 EventSourcedEntity EvaluationEntity holding state Evaluation{evaluationId,
  scenarioId, taskDescription, maxAttempts, EvaluationStatus status,
  List<TrajectoryAttempt> attempts, Optional<Integer> passedAttemptNumber,
  Optional<RecordedTrajectory> passedTrajectory, Optional<String> failureReason,
  Instant createdAt, Optional<Instant> finishedAt}. EvaluationStatus enum:
  RUNNING, EVALUATING, PASSED, FAILED_FINAL. Events: EvaluationCreated,
  TrajectoryRecorded, TrajectoryGuardrailVerdictRecorded, TrajectoryEvaluated,
  EvaluationPassed, EvaluationFailedFinal, EvalRecorded. Commands: createEvaluation,
  recordTrajectory, recordGuardrail, recordVerdict, pass, failFinal, recordEval,
  getEvaluation. emptyState() returns Evaluation.initial("", "", 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity ReferenceStore with command registerReference(scenarioId,
  steps) emitting ReferenceRegistered{scenarioId, steps, registeredAt} and
  command updateReference(scenarioId, steps) emitting ReferenceUpdated{scenarioId,
  steps, updatedAt}. The reference is keyed by scenarioId; the entity id is the
  scenarioId. getReference command returns Optional<ReferencePath>.

- 1 View EvaluationsView with row type EvaluationRow (mirrors Evaluation; the
  attempts list is preserved as-is — the list is bounded at maxAttempts so size
  stays reasonable). Table updater consumes EvaluationEntity events. ONE query
  getAllEvaluations SELECT * AS evaluations FROM evaluations_view. No WHERE status
  filter — caller filters client-side because Akka cannot auto-index enum columns
  (Lesson 2).

- 1 Consumer ScenarioConsumer subscribed to ReferenceStore events; on
  ScenarioSubmitted (produced by TrajectoryEndpoint via a submission event) starts
  a TrajectoryWorkflow with the evaluationId as the workflow id.

- 2 TimedActions:
  * ScenarioSimulator — every 60s, reads next line from
    src/main/resources/sample-events/task-scenarios.jsonl and calls
    TrajectoryEndpoint to submit the scenario (ensuring the reference path exists
    first by calling ReferenceStore.registerReference if absent).
  * EvalSampler — every 30s, queries EvaluationsView.getAllEvaluations, finds
    evaluations with a evaluated attempt that has not yet been recorded as an
    EvalRecorded event, and calls EvaluationEntity.recordEval(attemptNumber,
    verdict, deviationCount, stepLimitExceeded). Idempotent per (evaluationId,
    attemptNumber).

- 2 HttpEndpoints:
  * TrajectoryEndpoint at /api with POST /evaluations, GET /evaluations,
    GET /evaluations/{id}, GET /evaluations/sse, POST /references,
    GET /references/{scenarioId}, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /evaluations body is
    {scenarioId, taskDescription?, requestedBy?}; missing taskDescription defaults
    to the scenario's stored description, missing requestedBy defaults to "anonymous".
    Returns 404 if the scenarioId has no registered reference path.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- TrajectoryTasks.java declaring three Task<R> constants: EXECUTE (result
  conforms to RecordedTrajectory), RE_EXECUTE (RecordedTrajectory), EVALUATE
  (TrajectoryVerdict).
- Domain records ToolCall, RecordedTrajectory, GuardrailVerdict, Deviation,
  DeviationReport, TrajectoryVerdict, TrajectoryAttempt, ReferencePath, Evaluation;
  enums EvaluationStatus, EvalVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9235 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  trajectory-eval.runner.max-steps = 20,
  trajectory-eval.runner.max-attempts = 4 overridable by env var.
- src/main/resources/sample-events/task-scenarios.jsonl with 6 canned scenario
  lines, each shaped {"scenarioId":"...", "taskDescription":"...", "steps":[...]}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, E1 eval-periodic performance-monitor, HT1 halt
  graceful-degradation) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = agent-behavior-regression-testing,
  decisions.authority_level = advisory, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/runner.md, prompts/evaluator.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Trajectory Evaluation",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try it /
  How it works / Components / API contract cards); Architecture (4 mermaid
  diagrams + click-to-expand component table); Risk Survey (7 sub-tabs from
  governance.html with answers populated from risk-survey.yaml; unanswered .qb
  opacity 0.45); Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source
  table with click-to-expand rows); App UI (form + live list with status pills,
  click-to-expand per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Trajectory Evaluation</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name. Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (runner.json,
  evaluator.json), picks one entry pseudo-randomly per call, and deserialises
  it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    runner.json — 6 RecordedTrajectory entries. Three are well-formed
      trajectories of 4–8 steps that match common reference paths in
      task-scenarios.jsonl. Two are "adjusted" trajectories that reduce
      deviations from a prior report. One is an over-step-limit trajectory
      (>22 steps) used to exercise the guardrail in J3.
    evaluator.json — 6 TrajectoryVerdict entries. Three return verdict=PASS
      with deviationCount=0 and a one-sentence rationale. Three return
      verdict=FAIL with deviationCount=2 or 3 and a DeviationReport payload
      identifying mismatched steps.
- A MockModelProvider.seedFor(evaluationId, attemptNumber) helper makes the
  selection deterministic per (evaluationId, attemptNumber) so the same
  evaluation in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. RunnerAgent
  and EvaluatorAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a TrajectoryTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Evaluation row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: TrajectoryTasks.java is mandatory; generating RunnerAgent or
  EvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9235, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND theme variables for state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records only
  ${?VAR_NAME} substitution; Bootstrap.java fails fast if the reference does
  not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. The DOM contains exactly five <section class="tab-panel">
  elements; removed panels are deleted from the HTML, not hidden with display:none.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
