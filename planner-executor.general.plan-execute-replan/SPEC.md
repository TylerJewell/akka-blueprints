# SPEC — plan-execute-replan

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Plan-and-Execute Agent.
**One-line pitch:** Submit a goal; a Planner decomposes it into a numbered plan, an Executor carries out each step with a scoped tool set, and a Replanner revises the plan after every observation until the goal is reached or the budget is exhausted.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern in its three-agent form: Planner, Executor, Replanner. The Planner owns the initial `ExecutionPlan` (a numbered list of steps, each with a `ToolAssignment` and expected output shape). The Executor owns each individual step — it invokes the assigned tool and returns a `StepResult`. The Replanner owns the loop-level decision: given the goal, the current plan, and all accumulated `Observation` records, it produces a `ReplanDecision` — `Continue(nextStepIndex)`, `Revise(updatedPlan, reason)`, or `Conclude(conclusion)`.

Two budget limits govern the loop: a per-plan revision budget (at most three `Revise` decisions before the system emits `FAILED`), and a per-step retry budget (at most two consecutive failures on the same step before the Replanner is forced to `Revise` or `Fail`).

The blueprint wires two governance mechanisms into this loop:

- a **before-tool-call guardrail** that vets every `ToolAssignment` before the Executor runs the step, checking the tool is on the allow-list and the argument does not exceed the declared scope,
- a **replanner quality eval-event** that scores each `ReplanDecision` produced by the Replanner and records the score on the observation ledger, giving operators an auditable quality signal at the most consequential decision point in the loop.

## 3. User-facing flows

The user opens the App UI tab and submits a goal via the form.

1. The system creates a `Goal` record in `PLANNING` and starts an `ExecutionWorkflow`.
2. The Planner produces an `ExecutionPlan { steps: List<PlanStep>, goalSummary }` and emits `GoalPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Picks the current step from the plan.
   - The **before-tool-call guardrail** vets the `ToolAssignment`; on rejection the workflow records a `StepBlocked` observation and increments the step's failure count.
   - On approval, the Executor runs the step and returns a `StepResult`.
   - The workflow appends a `StepResult` as an `Observation` to the observation ledger.
   - The Replanner reads the goal, the plan, and all observations, then emits a `ReplanDecision`.
   - The **quality eval-event** scores the `ReplanDecision`; the score is appended as an `EvalRecord` on the observation ledger.
   - On `Continue`: advance to the next step index.
   - On `Revise`: replace the plan with the updated plan; emit `PlanRevised`; increment the revision count.
   - On `Conclude`: produce a `GoalConclusion { summary, citations }` and emit `GoalConcluded`.
4. If the revision budget or the step failure budget is exhausted, the workflow emits `GoalFailed`.
5. The operator can press **Pause execution** in the dashboard at any time. The in-flight step finishes; the next loop iteration exits with `GoalPaused`. The goal moves to `PAUSED`.

A `RequestSimulator` (TimedAction) drips a sample goal every 90 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Converts a user goal into a typed `ExecutionPlan`. | `ExecutionWorkflow` | returns `ExecutionPlan` to workflow |
| `ExecutorAgent` | `AutonomousAgent` | Runs a single `PlanStep` using the assigned tool; returns `StepResult`. | `ExecutionWorkflow` | returns `StepResult` to workflow |
| `ReplannerAgent` | `AutonomousAgent` | Reads goal, plan, and observations; returns `ReplanDecision`. | `ExecutionWorkflow` | returns `ReplanDecision` to workflow |
| `ExecutionWorkflow` | `Workflow` | Drives the plan → guard → execute → observe → replan loop, plus revise, conclude, and fail branches. | `GoalEndpoint`, `GoalRequestConsumer` | `GoalEntity` |
| `GoalEntity` | `EventSourcedEntity` | Holds the goal's lifecycle, current plan, observation ledger, eval records, and final conclusion. | `ExecutionWorkflow` | `GoalView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator pause flag. Single instance keyed by literal `"global"`. | `GoalEndpoint` (operator action) | `ExecutionWorkflow` (polls) |
| `GoalQueue` | `EventSourcedEntity` | Audit log of submitted goals. | `GoalEndpoint`, `RequestSimulator` | `GoalRequestConsumer` |
| `GoalView` | `View` | List-of-goals read model for the UI. | `GoalEntity` events | `GoalEndpoint` |
| `GoalRequestConsumer` | `Consumer` | Subscribes to `GoalQueue` events; starts an `ExecutionWorkflow` per submission. | `GoalQueue` events | `ExecutionWorkflow` |
| `RequestSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/goal-prompts.jsonl` and enqueues it. | scheduler | `GoalQueue` |
| `StuckGoalMonitor` | `TimedAction` | Every 30 s, marks any goal stuck in `EXECUTING` past 5 minutes as `STUCK`. | scheduler | `GoalEntity` |
| `GoalEndpoint` | `HttpEndpoint` | `/api/goals/*` — submit, get, list, SSE, operator pause/resume. | — | `GoalView`, `GoalQueue`, `GoalEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record GoalRequest(String goal, String requestedBy) {}

record PlanStep(
    int stepIndex,
    String description,
    ToolAssignment tool,
    String expectedOutput
) {}

record ToolAssignment(
    ToolKind kind,
    String argument
) {}

record ExecutionPlan(
    String goalSummary,
    List<PlanStep> steps
) {}

record StepResult(
    int stepIndex,
    ToolKind tool,
    boolean ok,
    String output,
    Optional<String> errorReason
) {}

record Observation(
    int stepIndex,
    ObservationType type,
    String content,
    Optional<EvalRecord> evalRecord,
    Instant recordedAt
) {}

record EvalRecord(
    String dimension,
    int score,
    String rationale
) {}

record ObservationLedger(List<Observation> entries) {}

record GoalConclusion(String summary, List<String> citations, Instant producedAt) {}

record Goal(
    String goalId,
    String goal,
    GoalStatus status,
    Optional<ExecutionPlan> plan,
    int currentStepIndex,
    int revisionCount,
    Optional<ObservationLedger> observations,
    Optional<GoalConclusion> conclusion,
    Optional<String> failureReason,
    Optional<String> pauseReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ToolKind { SEARCH, READ, CALCULATE, SUMMARISE }
enum ObservationType { STEP_OK, STEP_FAILED, STEP_BLOCKED, PLAN_REVISED, EVAL }
enum GoalStatus { PLANNING, EXECUTING, CONCLUDED, FAILED, PAUSED, STUCK }
```

### Events (`GoalEntity`)

`GoalCreated`, `GoalPlanned`, `StepDispatched`, `StepBlocked`, `StepObserved`, `PlanRevised`, `EvalRecorded`, `GoalConcluded`, `GoalFailed`, `GoalPaused`, `GoalFailedTimeout`.

### Events (`SystemControlEntity`)

`PauseRequested`, `PauseCleared`.

### Events (`GoalQueue`)

`GoalSubmitted { goalId, goal, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/goals` — body `{ goal, requestedBy? }` → `202 { goalId }`. Starts a workflow.
- `GET /api/goals` — list all goals. Optional `?status=...`.
- `GET /api/goals/{id}` — one goal (full plan + observations + conclusion).
- `GET /api/goals/sse` — server-sent events stream of every goal change.
- `POST /api/control/pause` — body `{ reason }` → `200`. Sets the operator pause flag.
- `POST /api/control/resume` — `200`. Clears the operator pause flag.
- `GET /api/control` — `{ paused, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Plan-and-Execute Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a goal, operator pause/resume control, live list of goals with status pills, expand-row to see the plan steps, observations, eval records, and the final conclusion.

Browser title: `<title>Akka Sample: Plan-and-Execute Agent</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `ExecutorAgent`): every `ToolAssignment` is vetted before the Executor runs the step. Checks (a) the tool is in `ToolKind` — `SEARCH`, `READ`, `CALCULATE`, `SUMMARISE`; (b) `SEARCH` arguments do not reference disallowed hosts; (c) `READ` arguments stay inside `sample-data/`; (d) `CALCULATE` expressions do not contain `eval`-shaped patterns. Blocking. Rejection → `StepBlocked` observation + incremented failure count; the Replanner sees the block on its next call.
- **E1 — replanner quality eval-event** (`eval-event`, flavor `on-decision-eval`): after every `ReplanDecision` the workflow runs `ReplanQualityEvaluator.score(decision, observations)`. The evaluator checks alignment between the revised plan's steps and the goal summary, absence of repeated failed steps, and whether the `Conclude` decision is premature. The score (0–100) and a one-sentence rationale are stored as an `EvalRecord` appended to the observation ledger. Non-blocking; the score is informational but visible to operators in the UI.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Converts goal to `ExecutionPlan`.
- `ExecutorAgent` → `prompts/executor.md`. Runs a single `PlanStep`.
- `ReplannerAgent` → `prompts/replanner.md`. Produces `ReplanDecision` from accumulated observations.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Research the current Akka SDK version, find its release date, and produce a three-sentence summary." Goal progresses `PLANNING → EXECUTING → CONCLUDED` within ~3 minutes. The expanded view shows a plan with 3–5 steps, an observation ledger with one entry per step, at least one `EvalRecord`, and a non-empty `GoalConclusion`.
2. **J2** — Submit a goal whose first plan step assigns `READ` with an argument outside `sample-data/`. The guardrail blocks the step; the replanner revises; the goal either concludes via an alternate path or fails after the revision budget is exhausted.
3. **J3** — Submit any goal and click **Pause execution** while it is `EXECUTING`. The in-flight step finishes; the next loop iteration checks the pause flag and emits `GoalPaused`; the goal moves to `PAUSED`.
4. **J4** — Submit a goal that forces the Replanner to produce a low-quality revision (repeat of a failed step). The `EvalRecord` shows a score below 40; the operator can see the rationale in the expanded observation row.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named plan-execute-replan demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-plan-execute-replan. Java
package io.akka.samples.planandexecuteagent. Akka 3.6.0. HTTP port 9814.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PlannerAgent — definition() with one capability:
      capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN returns ExecutionPlan.
  * ExecutorAgent — capability(TaskAcceptance.of(EXECUTE_STEP).maxIterationsPerTask(2)).
    Prompt from prompts/executor.md. Returns StepResult.
  * ReplannerAgent — capability(TaskAcceptance.of(REPLAN).maxIterationsPerTask(2)).
    Prompt from prompts/replanner.md. Returns ReplanDecision — a sealed interface
    with permits Continue(nextStepIndex), Revise(ExecutionPlan updatedPlan, String reason),
    Conclude(GoalConclusion conclusion).

- 1 Workflow ExecutionWorkflow with steps:
  planStep -> [loop entry] checkPauseStep -> guardStep -> executeStep ->
  observeStep -> replanStep -> evalStep -> decideStep
  -> [back to checkPauseStep or to concludeStep / failStep / pausedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), executeStep ofSeconds(90),
    replanStep ofSeconds(45), evalStep ofSeconds(30), concludeStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(ExecutionWorkflow::error)).
  checkPauseStep reads SystemControlEntity.get; on paused=true transitions to
  pausedStep (emits GoalPaused on GoalEntity).
  guardStep runs ToolGuardrail.vet(ToolAssignment); on reject records a
  StepBlocked observation via GoalEntity.recordBlock(stepIndex, reason) and
  loops back to replanStep (increments the step failure count, skipping the
  execute and observe steps for this iteration).
  executeStep calls forAutonomousAgent(ExecutorAgent.class, EXECUTE_STEP)
  passing the current PlanStep, then calls GoalEntity.observeStep(result).
  observeStep calls GoalEntity.recordObservation(observation).
  replanStep calls forAutonomousAgent(ReplannerAgent.class, REPLAN).
  evalStep runs ReplanQualityEvaluator.score(replanDecision, observations)
  and calls GoalEntity.recordEval(evalRecord).
  decideStep switches on ReplanDecision: Continue → advance stepIndex and
  loop; Revise → call GoalEntity.revise(updatedPlan); increment revisionCount;
  loop if revisionCount < 3, else transition to failStep; Conclude →
  transition to concludeStep.

- 1 EventSourcedEntity GoalEntity holding Goal state. emptyState() returns
  Goal.initial("", null). Commands: createGoal, recordPlan, recordBlock,
  recordObservation, revisePlan, recordEval, concludeGoal, failGoal,
  pauseGoal, timeoutFail, getGoal. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean paused, Optional<String> reason, Optional<Instant>
  pausedAt}. Commands: requestPause(reason), clearPause, get. Events:
  PauseRequested, PauseCleared.

- 1 EventSourcedEntity GoalQueue with command enqueueGoal(goalId, goal,
  requestedBy) emitting GoalSubmitted.

- 1 View GoalView with row type GoalRow (mirror of Goal minus heavy
  observation payloads — truncate to last 3 observations plus counts;
  the UI fetches the full goal by id on click). Table updater consumes
  GoalEntity events. ONE query getAllGoals SELECT * AS goals FROM goal_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer GoalRequestConsumer subscribed to GoalQueue events; on
  GoalSubmitted starts an ExecutionWorkflow with goalId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 90s, reads next line from
    src/main/resources/sample-events/goal-prompts.jsonl and calls
    GoalQueue.enqueueGoal.
  * StuckGoalMonitor — every 30s, queries GoalView.getAllGoals, filters
    EXECUTING goals whose createdAt is older than 5 minutes, calls
    GoalEntity.timeoutFail; ExecutionWorkflow polls GoalEntity.getGoal in its
    decideStep and exits when status == STUCK.

- 2 HttpEndpoints:
  * GoalEndpoint at /api with POST /goals, GET /goals (filters client-side
    from getAllGoals), GET /goals/{id}, GET /goals/sse,
    POST /control/pause, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring Task<R> constant: PLAN (resultConformsTo
  ExecutionPlan).
- ExecutorTasks.java declaring Task<R> constant: EXECUTE_STEP (resultConformsTo
  StepResult).
- ReplannerTasks.java declaring Task<R> constant: REPLAN (resultConformsTo
  ReplanDecision).
- Domain records as listed in SPEC §5, plus a ReplanDecision sealed interface
  with permits Continue(int nextStepIndex), Revise(ExecutionPlan updatedPlan,
  String reason), Conclude(GoalConclusion conclusion).
- application/ToolGuardrail.java — deterministic vetter. Reject if
  ToolKind is not in {SEARCH, READ, CALCULATE, SUMMARISE}; if SEARCH
  argument names a host not on the allow-list (akka.io, doc.akka.io,
  github.com); if READ argument is not under sample-data/; if CALCULATE
  argument matches /\b(eval|exec|import|__)\b/.
- application/ReplanQualityEvaluator.java — deterministic scorer (0–100).
  Deducts 30 points if the revised plan repeats a step whose ObservationType
  is STEP_FAILED more than once; deducts 20 points if a Conclude is produced
  when fewer than half the original plan steps have observations; deducts 15
  points per step whose ToolAssignment.kind differs from the original plan
  for that index without a blocker justification. Stores rationale as a
  single sentence.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9814 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/goal-prompts.jsonl with 8 canned goal
  prompts spanning research, calculation, file inspection, and summarisation.
- src/main/resources/sample-data/ — 8 short fixture files used by READ
  tool simulations.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, E1)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance; deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/executor.md, prompts/replanner.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Plan-and-Execute
  Agent", one-line pitch, prerequisites (integration form host-software
  requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator pause/resume control + live list with
  status pills and expand-on-click for plan, observations, eval records,
  and conclusion). Browser title exactly:
  <title>Akka Sample: Plan-and-Execute Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE; the value lives in the user's infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json.
- Per-agent mock-response shapes:
    planner.json — 4–6 ExecutionPlan entries; each has 3–5 PlanStep records
      with ToolKind values spanning SEARCH, READ, CALCULATE, SUMMARISE.
    executor.json — 6 StepResult entries; ok=true for most; one entry has
      ok=false with errorReason="no fixture matched the argument".
    replanner.json — a sectioned file with three lists keyed by decision type:
      "CONTINUE" → 4 Continue(nextStepIndex) entries incrementing step index.
      "REVISE" → 3 Revise entries with updated plans and one-line reasons.
      "CONCLUDE" → 4 Conclude entries with GoalConclusion payloads
        (60–100 word summaries, 3–4 citation bullets).
- A MockModelProvider.seedFor(goalId) helper makes the selection deterministic
  per goal id across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent — the
  base class clause must read `extends AutonomousAgent` for every agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, executeStep, replanStep, evalStep,
  concludeStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the Goal entity state (plan, observations, conclusion, failureReason,
  pauseReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java,
  ExecutorTasks.java, ReplannerTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9814 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of
  the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM — delete
  removed tabs, do not display:none them.
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
