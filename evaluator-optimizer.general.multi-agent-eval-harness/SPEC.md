# SPEC — multi-agent-eval-harness

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Multi-Agent Evaluation Harness.
**One-line pitch:** Submit a scenario suite; an orchestrator agent fans work out to specialist agents; a judge agent scores the aggregate result; the harness records every outcome, blocks CI promotion on failure, and continuously samples eval events for quality trending.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to quality measurement: a Workflow loads a scenario suite, dispatches each scenario to an `OrchestratorAgent` that coordinates specialist execution, then hands the aggregate results to a `JudgeAgent` that produces a structured pass/fail verdict against a rubric. Two governance controls ride alongside the core loop — a **ci-gate** that blocks deployment promotion when the aggregate score falls below a configured threshold, and an **eval-periodic** recorder that captures per-scenario quality signals on a continuous schedule for trend analysis. The halt mechanism ends the workflow gracefully when the overall verdict is `FAIL` after exhausting retries.

## 3. User-facing flows

The user opens the App UI tab and submits an eval run (a scenario suite name and an optional pass threshold override).

1. The system creates an `EvalRun` record in `RUNNING` and starts an `EvalRunWorkflow`.
2. The workflow loads scenario definitions from `ScenarioRegistry` for the named suite.
3. The `OrchestratorAgent` receives the scenario set, dispatches each one to the appropriate specialist execution path, and returns an `AggregateResult` containing per-scenario `ScenarioResult` records.
4. A deterministic threshold check verifies the aggregate score is at or above `passThreshold`. Runs below threshold are flagged for the CI gate before judgment.
5. The `JudgeAgent` scores the `AggregateResult` against a rubric (coverage, accuracy, coherence, safety) and returns either `PASS` with an overall score, or `FAIL` with a `JudgmentNotes` payload (up to four bullets identifying weak scenarios).
6. On `PASS`, the workflow transitions the run to `PASSED`, emits a `RunPassed` event, and records the aggregate score.
7. On `FAIL`, the workflow checks whether any retry attempts remain. If retries remain, the orchestrator re-runs only the failing scenarios. If the retry ceiling is reached, the workflow ends with `FAILED`, preserving all scenario results and every judgment for audit.
8. On every terminal transition, the CI gate control evaluates the aggregate score against the configured threshold and emits a `CIGateBlocked` event if the threshold is not met — even for a `PASS` that narrowly missed being blocked.

A `ScenarioLoader` (TimedAction) submits a canned eval run every 120 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OrchestratorAgent` | `AutonomousAgent` | Dispatches scenarios to specialist paths; collects raw responses into an `AggregateResult`. | `EvalRunWorkflow` | returns `AggregateResult` to workflow |
| `JudgeAgent` | `AutonomousAgent` | Scores `AggregateResult` against the rubric; returns `PASS` or `FAIL` with notes. | `EvalRunWorkflow` | returns `Judgment` to workflow |
| `EvalRunWorkflow` | `Workflow` | Loads scenarios → orchestrates → threshold-checks → judges → halts at retry ceiling. | `EvalEndpoint`, `ScenarioResultConsumer` | `EvalRunEntity` |
| `EvalRunEntity` | `EventSourcedEntity` | Holds the run lifecycle, every scenario result, every judgment, and the final outcome. | `EvalRunWorkflow` | `EvalRunsView` |
| `ScenarioRegistry` | `EventSourcedEntity` | Stores scenario definitions; queried by the workflow at run start. | `EvalEndpoint` | `EvalRunWorkflow` |
| `EvalRunsView` | `View` | List-of-runs read model. | `EvalRunEntity` events | `EvalEndpoint` |
| `ScenarioResultConsumer` | `Consumer` | Subscribes to `ScenarioRegistry` events; starts a workflow per submitted run. | `ScenarioRegistry` events | `EvalRunWorkflow` |
| `ScenarioLoader` | `TimedAction` | Drips a sample eval run every 120 s from `sample-events/scenario-suites.jsonl`. | scheduler | `ScenarioRegistry` |
| `EvalSampler` | `TimedAction` | Every 60 s, scans `EvalRunsView`, records a `ScenarioEvalRecorded` event for any scenario result that has been judged but not yet sampled. | scheduler | `EvalRunEntity` |
| `EvalEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `EvalRunsView`, `ScenarioRegistry`, `EvalRunEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record ScenarioSpec(
    String scenarioId,
    String suiteName,
    String prompt,
    String expectedBehavior,
    String category
) {}

record ScenarioResult(
    String scenarioId,
    String agentResponse,
    boolean thresholdMet,
    Instant completedAt
) {}

record AggregateResult(
    String suiteName,
    List<ScenarioResult> scenarioResults,
    double aggregateScore,
    Instant aggregatedAt
) {}

record JudgmentNotes(List<String> bullets, String overallRationale) {}

record Judgment(
    JudgeVerdict verdict,
    JudgmentNotes notes,
    double score,
    Instant evaluatedAt
) {}

record EvalRun(
    String runId,
    String suiteName,
    double passThreshold,
    int maxRetries,
    RunStatus status,
    List<ScenarioResult> scenarioResults,
    List<Judgment> judgments,
    Optional<Double> finalScore,
    Optional<String> failureReason,
    boolean ciGateBlocked,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RunStatus { RUNNING, JUDGING, PASSED, FAILED }

enum JudgeVerdict { PASS, FAIL }
```

### Events (on `EvalRunEntity`)

`RunCreated`, `ScenariosLoaded`, `OrchestratorResultRecorded`, `JudgmentRecorded`, `RunPassed`, `RunFailed`, `CIGateBlocked`, `ScenarioEvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ suiteName, passThreshold?, requestedBy? }` → `{ runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=RUNNING|JUDGING|PASSED|FAILED`.
- `GET /api/runs/{id}` — one run (including every scenario result and every judgment).
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Multi-Agent Evaluation Harness"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, ci-gate = red).
- **App UI** — form to submit a run (suite name + pass threshold), live list of runs with status pills, click-to-expand per-scenario timeline showing each agent response, the threshold verdict, the judge's verdict, and the judgment notes.

Browser title: `<title>Akka Sample: Multi-Agent Evaluation Harness</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-periodic** (`continuous-accuracy`): every scenario result is recorded as a `ScenarioEvalRecorded` event with `{ scenarioId, verdict, score, thresholdMet }`. The `EvalSampler` TimedAction is the canonical writer; the workflow also emits an event on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-run timeline and in `/api/runs/{id}`.
- **CI1 — ci-gate** (`test-gate`): after every terminal transition, the aggregate score is compared against the configured `passThreshold`. If the score falls below threshold, a `CIGateBlocked` event is emitted and a `ciGateBlocked = true` flag is set on the run. An external CI pipeline consuming the SSE stream or polling `/api/runs/{id}` can treat this flag as the gate signal. Enforcement: build-gate.

## 9. Agent prompts

- `OrchestratorAgent` → `prompts/orchestrator.md`. Dispatches scenarios to appropriate specialist execution paths; collects and normalises per-scenario responses into an `AggregateResult`.
- `JudgeAgent` → `prompts/judge.md`. Scores an `AggregateResult` against a fixed rubric; returns `PASS` with an overall score or `FAIL` with up to four short bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — passing run** — Submit a suite; run progresses `RUNNING` → `JUDGING` → `PASSED`; the App UI shows every scenario result and the judgment.
2. **J2 — failing run** — Submit a suite with a threshold set above what the agents can achieve; run reaches `FAILED` after retries with the failure reason preserved.
3. **J3 — CI gate fires** — A run that lands in `FAILED` (or a near-miss `PASSED`) emits a `CIGateBlocked` event visible in the timeline; `ciGateBlocked` is `true` on `GET /api/runs/{id}`.
4. **J4 — eval-event timeline** — The expanded view of any completed run shows one `ScenarioEvalRecorded` event per judged scenario.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-agent-eval-harness demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-multi-agent-eval-harness.
Java package io.akka.samples.multiagentevaluation. Akka 3.6.0. HTTP port 9784.

Components to wire (exactly):
- 2 AutonomousAgents:
  * OrchestratorAgent — definition() with
    capability(TaskAcceptance.of(DISPATCH_SCENARIOS).maxIterationsPerTask(5))
    AND capability(TaskAcceptance.of(RERUN_FAILING).maxIterationsPerTask(3)).
    System prompt loaded from prompts/orchestrator.md. Returns AggregateResult{suiteName,
    scenarioResults, aggregateScore, aggregatedAt} for both DISPATCH_SCENARIOS and
    RERUN_FAILING. The RERUN_FAILING task takes (originalSuite, failingScenarioIds,
    priorJudgmentNotes) as inputs.
  * JudgeAgent — definition() with
    capability(TaskAcceptance.of(JUDGE_RUN).maxIterationsPerTask(2)). System
    prompt from prompts/judge.md. Returns Judgment{verdict, notes, score,
    evaluatedAt} where verdict is the JudgeVerdict enum (PASS | FAIL)
    and score is a 0.0–1.0 double rubric.

- 1 Workflow EvalRunWorkflow with steps:
    startStep -> loadScenariosStep -> dispatchStep -> thresholdStep ->
    [threshold FAIL before judging? retryOrFail : judgeStep] ->
    [verdict PASS? passStep : (retriesRemaining > 0 ?
       rerunStep with failing scenario ids : failStep)] ->
    ciGateStep -> END.
  dispatchStep calls forAutonomousAgent(OrchestratorAgent.class, runId)
    .runSingleTask(DISPATCH_SCENARIOS or RERUN_FAILING).
  judgeStep calls forAutonomousAgent(JudgeAgent.class, runId).runSingleTask(JUDGE_RUN).
  thresholdStep is a pure-function step (no LLM call): checks
    aggregateResult.aggregateScore() >= passThreshold. On FAIL emits
    CIGateBlocked event and continues to judgeStep regardless.
  ciGateStep is a pure-function step: emits CIGateBlocked when
    finalScore < passThreshold and ciGateBlocked has not already been set.
  passStep emits RunPassed with finalScore and accepted judgment.
  failStep emits RunFailed with failureReason = "max retries reached (N)"
    plus the best-scoring judgment's rationale. Override settings()
    with stepTimeout(90s) on dispatchStep and judgeStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  maxRetries read from multi-agent-eval.run.max-retries (default 2).

- 1 EventSourcedEntity EvalRunEntity holding state EvalRun{runId, suiteName,
  passThreshold, maxRetries, RunStatus status, List<ScenarioResult> scenarioResults,
  List<Judgment> judgments, Optional<Double> finalScore, Optional<String> failureReason,
  boolean ciGateBlocked, Instant createdAt, Optional<Instant> finishedAt}.
  RunStatus enum: RUNNING, JUDGING, PASSED, FAILED. Events: RunCreated,
  ScenariosLoaded, OrchestratorResultRecorded, JudgmentRecorded, RunPassed,
  RunFailed, CIGateBlocked, ScenarioEvalRecorded. Commands: createRun,
  recordScenariosLoaded, recordOrchestratorResult, recordJudgment, pass,
  fail, recordCIGate, recordScenarioEval, getRun. emptyState() returns
  EvalRun.initial() with no commandContext() reference. Event-applier
  wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity ScenarioRegistry with command registerSuite(suiteName,
  scenarios) emitting SuiteRegistered{registryId, suiteName, scenarios, registeredAt}.
  Also command getSuite(suiteName) returning List<ScenarioSpec>.

- 1 View EvalRunsView with row type EvalRunRow (mirrors EvalRun; the scenarioResults
  list is bounded at the number of scenarios in the suite so size stays reasonable).
  Table updater consumes EvalRunEntity events. ONE query getAllRuns SELECT *
  AS runs FROM eval_runs_view. No WHERE status filter — caller filters
  client-side (Lesson 2).

- 1 Consumer ScenarioResultConsumer subscribed to ScenarioRegistry events; on
  SuiteRegistered starts an EvalRunWorkflow with the runId as the workflow id.

- 2 TimedActions:
  * ScenarioLoader — every 120s, reads next line from
    src/main/resources/sample-events/scenario-suites.jsonl and calls
    ScenarioRegistry.registerSuite.
  * EvalSampler — every 60s, queries EvalRunsView.getAllRuns, finds runs
    with a judged scenario that has not yet been recorded as a
    ScenarioEvalRecorded event, and calls EvalRunEntity.recordScenarioEval(
    scenarioId, verdict, score, thresholdMet). Idempotent per (runId, scenarioId).

- 2 HttpEndpoints:
  * EvalEndpoint at /api with POST /runs, GET /runs, GET /runs/{id},
    GET /runs/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /runs body
    is {suiteName, passThreshold?, requestedBy?}; missing passThreshold
    defaults to 0.75, missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- EvalTasks.java declaring three Task<R> constants: DISPATCH_SCENARIOS (resultConformsTo
  AggregateResult), RERUN_FAILING (AggregateResult), JUDGE_RUN (Judgment).
- Domain records ScenarioSpec, ScenarioResult, AggregateResult, JudgmentNotes,
  Judgment, EvalRun; enums RunStatus, JudgeVerdict.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9784 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-run config
  multi-agent-eval.run.max-retries = 2 and
  multi-agent-eval.run.default-pass-threshold = 0.75, overridable by env var.
- src/main/resources/sample-events/scenario-suites.jsonl with 6 canned suite
  lines, each shaped {"suiteName":"...", "passThreshold":0.75}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-periodic
  continuous-accuracy, CI1 ci-gate test-gate) and a matching simplified_view
  list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = multi-agent-quality-measurement,
  decisions.authority_level = advisory, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/orchestrator.md, prompts/judge.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Evaluation Harness",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-scenario timeline). Browser title exactly:
  <title>Akka Sample: Multi-Agent Evaluation Harness</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: orchestrator.json, judge.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    orchestrator.json — 6 AggregateResult entries. Three are successful
      orchestrations with aggregateScore between 0.80 and 0.95 and 4-6
      ScenarioResult records each with thresholdMet=true. Two are partial
      results with aggregateScore between 0.55 and 0.70 and some
      thresholdMet=false scenarios. One is an orchestration result with
      aggregateScore below 0.40 used to exercise the CI gate in J3.
    judge.json — 6 Judgment entries. Three return verdict=PASS with
      score=0.85 or 0.92 and a one-sentence rationale. Three return
      verdict=FAIL with score=0.45 or 0.60 and a JudgmentNotes payload
      of four bullets ("scenario S-3 response was off-topic",
      "scenario S-5 lacked required citations",
      "coherence score below 0.5 on two scenarios",
      "safety rubric failed on S-7").
- A MockModelProvider.seedFor(runId, attemptNumber) helper makes the
  selection deterministic per (runId, attemptNumber) so the same run in
  dev produces the same trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. OrchestratorAgent
  and JudgeAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with an EvalTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(90s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the EvalRun row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: EvalTasks.java is mandatory; generating OrchestratorAgent or
  JudgeAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9784, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
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
