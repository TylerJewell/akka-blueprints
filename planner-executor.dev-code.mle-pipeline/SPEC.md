# SPEC — mle-pipeline

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** ML Engineering Pipeline.
**One-line pitch:** Submit a dataset and objective; a Planner stages the ML pipeline on a pipeline ledger, dispatches each stage to one of four specialist agents (profiler, feature engineer, trainer, evaluator), records outcomes on an evaluation ledger, gates model promotion on quality metrics, and replans when gates fail.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern applied to multi-stage ML model development. The Planner owns two ledgers — a **pipeline ledger** (dataset facts known, gaps still open, ordered stages, active dispatch) and an **evaluation ledger** (each stage's attempt count, quality verdict, gate outcome, observed metrics). On each loop tick the Planner reads both ledgers, picks the next specialist, and emits one of four decisions: `CONTINUE`, `REPLAN`, `PROMOTE`, or `FAIL`. On two consecutive gate failures without progress, the Planner emits a terminal `FAIL`.

The blueprint demonstrates two governance controls wired into that loop:

- a **build-gate CI control** (eval-gate) that must pass before a model advances to the promotion stage — if metric thresholds are not met the gate blocks and the Planner must replan with different hyperparameters or a different model family,
- a **periodic drift and fairness watch** (eval-periodic) that fires outside the main loop on a timed schedule, compares the latest model's metric bundle against baseline, and raises a `DriftAlertRaised` event when drift exceeds the configured threshold or fairness metrics diverge.

## 3. User-facing flows

The user opens the App UI tab and submits a pipeline run via the form.

1. The system creates a `PipelineRun` record in `PLANNING` and starts a `PipelineWorkflow`.
2. The Planner drafts a `PipelineLedger { facts, gaps, stages, activeDispatch }` and emits `RunPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads both ledgers and proposes a `StageDispatch { specialist, stageName, instruction, rationale }`.
   - The **eval-gate guardrail** intercepts dispatches to `PROMOTE` stage: it checks the evaluation ledger for a passing `ModelEvaluatorAgent` verdict before allowing promotion to proceed. On gate failure the workflow records a `StageGateFailed` entry and asks the Planner to replan.
   - The chosen specialist runs the stage and returns a typed `StageResult`.
   - The workflow appends an `EvalEntry { specialist, stageName, attempt, verdict, metrics, gateOutcome, recordedAt }` to the evaluation ledger.
4. The Planner decides on each loop tick: `CONTINUE`, `REPLAN`, `PROMOTE`, or `FAIL`. After two consecutive gate failures or three consecutive `REPLAN` outputs without progress, the Planner emits `FAIL`.
5. On `PROMOTE`, the Planner produces a `ModelReport { summary, metricBundle, stages }` and emits `RunCompleted`. The PipelineRun moves to `COMPLETED`.
6. The operator can press **Halt new dispatches** in the dashboard at any time. The workflow finishes the in-flight stage, then ends with `RunHaltedOperator`. The PipelineRun moves to `HALTED`.
7. `DriftWatchMonitor` (TimedAction) runs every 5 minutes. It retrieves the evaluation ledger of completed runs, compares metrics against the baseline stored in the fixture, and emits `DriftAlertRaised` when drift or fairness thresholds are exceeded.

A `RunSimulator` (TimedAction) drips a sample pipeline request every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PlannerAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains pipeline ledger; reads evaluation ledger. Produces `ModelReport` on promotion. | `PipelineWorkflow` | returns typed result to workflow |
| `DataProfilerAgent` | `AutonomousAgent` | Returns dataset statistics and schema validation from seeded fixtures (`sample-data/datasets/`). | `PipelineWorkflow` | — |
| `FeatureEngineerAgent` | `AutonomousAgent` | Proposes feature transforms and encoding strategies; returns a `FeaturePlan`. | `PipelineWorkflow` | — |
| `ModelTrainerAgent` | `AutonomousAgent` | Returns training metadata (hyperparameters tried, convergence) from seeded fixtures. | `PipelineWorkflow` | — |
| `ModelEvaluatorAgent` | `AutonomousAgent` | Scores a trained model against eval sets; returns a `MetricBundle` with accuracy, F1, AUC, and fairness metrics. | `PipelineWorkflow` | — |
| `PipelineWorkflow` | `Workflow` | Drives the plan → dispatch-gated → execute → record → decide loop, plus replan and halt branches. | `PipelineEndpoint`, `RunRequestConsumer` | `PipelineRunEntity` |
| `PipelineRunEntity` | `EventSourcedEntity` | Holds the run's lifecycle, pipeline ledger, evaluation ledger, and final report. | `PipelineWorkflow` | `PipelineRunView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `PipelineEndpoint` (operator action) | `PipelineWorkflow` (polls) |
| `RunQueue` | `EventSourcedEntity` | Audit log of submitted pipeline runs. | `PipelineEndpoint`, `RunSimulator` | `RunRequestConsumer` |
| `PipelineRunView` | `View` | List-of-runs read model for the UI. | `PipelineRunEntity` events | `PipelineEndpoint` |
| `RunRequestConsumer` | `Consumer` | Subscribes to `RunQueue` events; starts a `PipelineWorkflow` per submission. | `RunQueue` events | `PipelineWorkflow` |
| `RunSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/run-prompts.jsonl` and enqueues it. | scheduler | `RunQueue` |
| `StaleRunMonitor` | `TimedAction` | Every 60 s, marks any run stuck in `EXECUTING` past 10 minutes as `STUCK`. The workflow polls this and ends with `RunFailedTimeout`. | scheduler | `PipelineRunEntity` |
| `DriftWatchMonitor` | `TimedAction` | Every 5 min, reads completed-run evaluation ledgers, compares metrics to baseline, emits `DriftAlertRaised` when thresholds exceeded. | scheduler | `PipelineRunEntity` |
| `PipelineEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, get, list, SSE, operator halt. Also `/api/alerts` for drift alerts. | — | `PipelineRunView`, `RunQueue`, `PipelineRunEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record RunRequest(String datasetRef, String objective, String requestedBy) {}

record PipelineLedger(
    List<String> facts,
    List<String> gaps,
    List<String> stages,
    Optional<StageDispatch> activeDispatch
) {}

record StageDispatch(
    SpecialistKind specialist,
    String stageName,
    String instruction,
    String rationale
) {}

record MetricBundle(
    double accuracy,
    double f1,
    double auc,
    double fairnessDelta,
    Optional<String> notes
) {}

record StageResult(
    SpecialistKind specialist,
    String stageName,
    boolean ok,
    String content,
    Optional<MetricBundle> metrics,
    Optional<String> errorReason
) {}

record EvalEntry(
    int attempt,
    SpecialistKind specialist,
    String stageName,
    EvalVerdict verdict,
    Optional<MetricBundle> metrics,
    GateOutcome gateOutcome,
    Optional<String> blocker,
    Instant recordedAt
) {}

record EvaluationLedger(List<EvalEntry> entries) {}

record ModelReport(
    String summary,
    MetricBundle finalMetrics,
    List<String> stages,
    Instant producedAt
) {}

record DriftAlert(
    String runId,
    String metricName,
    double baseline,
    double observed,
    double delta,
    Instant detectedAt
) {}

record PipelineRun(
    String runId,
    String datasetRef,
    String objective,
    RunStatus status,
    Optional<PipelineLedger> ledger,
    Optional<EvaluationLedger> evalLedger,
    Optional<ModelReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    List<DriftAlert> driftAlerts,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SpecialistKind { DATA_PROFILER, FEATURE_ENGINEER, MODEL_TRAINER, MODEL_EVALUATOR }
enum EvalVerdict { OK, GATE_FAILED, FAILED, BLOCKED }
enum GateOutcome { PASSED, FAILED, NOT_APPLICABLE }
enum RunStatus { PLANNING, EXECUTING, COMPLETED, GATE_FAILED, FAILED, HALTED, STUCK }
```

### Events (`PipelineRunEntity`)

`RunCreated`, `RunPlanned`, `StageDispatched`, `StageGateFailed`, `StageRecorded`, `LedgerRevised`, `RunCompleted`, `RunFailed`, `RunHaltedAutomatic`, `RunHaltedOperator`, `RunFailedTimeout`, `DriftAlertRaised`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`RunQueue`)

`RunSubmitted { runId, datasetRef, objective, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ datasetRef, objective, requestedBy? }` → `202 { runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=...`.
- `GET /api/runs/{id}` — one run (full ledgers + report).
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `GET /api/alerts` — list all drift alerts across all runs, newest first.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "ML Engineering Pipeline"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a run, operator halt/resume control, live list of runs with status pills, expand-row to see the pipeline ledger, the evaluation ledger entries, drift alerts, and the final model report.

Browser title: `<title>Akka Sample: ML Engineering Pipeline</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — eval-gate (build-gate)** (`ci-gate`, flavor `test-gate`): before the workflow dispatches a `PROMOTE` stage, it checks the evaluation ledger for a `MODEL_EVALUATOR` entry with `gateOutcome = PASSED` and all metric thresholds satisfied (accuracy ≥ 0.80, fairnessDelta ≤ 0.05). Failure blocks promotion and records a `StageGateFailed` entry; the Planner must replan. Blocking.
- **E1 — drift and fairness watch** (`eval-periodic`, flavor `drift-fairness-watch`): `DriftWatchMonitor` runs every 5 minutes. It compares each completed run's `MetricBundle` against the baseline in `sample-data/baselines/metrics-baseline.json`. When `|observed - baseline| > threshold` for any metric, or `fairnessDelta > 0.05`, it emits `DriftAlertRaised` on the `PipelineRunEntity`. Non-blocking — the alert is recorded; the run is not halted. The UI surfaces drift alerts with a warning badge.

## 9. Agent prompts

- `PlannerAgent` → `prompts/planner.md`. Maintains both ledgers; decides next stage.
- `DataProfilerAgent` → `prompts/data-profiler.md`. Returns dataset statistics.
- `FeatureEngineerAgent` → `prompts/feature-engineer.md`. Returns feature plans.
- `ModelTrainerAgent` → `prompts/model-trainer.md`. Returns training metadata.
- `ModelEvaluatorAgent` → `prompts/model-evaluator.md`. Returns metric bundles.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Profile the credit-risk dataset, engineer features, train a gradient-boosted classifier, evaluate, and promote." Run progresses `PLANNING → EXECUTING → COMPLETED` within ~5 minutes. UI reflects each transition via SSE. The expanded view shows a pipeline ledger with a non-empty stages list, an evaluation ledger with 4–8 entries, and a non-empty `ModelReport`.
2. **J2** — Submit a run configured to produce a model that fails accuracy threshold. The eval-gate blocks promotion; the Planner replans with adjusted hyperparameters; the run either passes on the next attempt or ends in `GATE_FAILED`.
3. **J3** — Submit a run and click **Halt new dispatches** while it is `EXECUTING`. The in-flight stage finishes; no further dispatches occur; the run ends in `HALTED`.
4. **J4** — `DriftWatchMonitor` compares a completed run's metrics against baseline and detects a 12% drop in AUC. A `DriftAlertRaised` event is recorded; the UI shows a warning badge on the run card and the alert appears in `GET /api/alerts`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mle-pipeline demonstrating the
planner-executor × dev-code cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-dev-code-mle-pipeline. Java
package io.akka.samples.machinelearningengineering. Akka 3.6.0. HTTP port 9762.

Components to wire (exactly):
- 5 AutonomousAgents:
  * PlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_PIPELINE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE_STAGE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/planner.md. PLAN_PIPELINE returns PipelineLedger.
    DECIDE_STAGE returns a NextStage tagged union (Continue(StageDispatch) |
    Replan(revisedLedger) | Promote(ModelReport stub) | Fail(reason)).
    COMPOSE_REPORT returns ModelReport.
  * DataProfilerAgent — capability(TaskAcceptance.of(PROFILE_DATA).maxIterationsPerTask(2)).
    Prompt from prompts/data-profiler.md. Returns StageResult.
  * FeatureEngineerAgent — capability(TaskAcceptance.of(ENGINEER_FEATURES).maxIterationsPerTask(2)).
    Prompt from prompts/feature-engineer.md. Returns StageResult with content
    holding a FeaturePlan-shaped JSON string.
  * ModelTrainerAgent — capability(TaskAcceptance.of(TRAIN_MODEL).maxIterationsPerTask(2)).
    Prompt from prompts/model-trainer.md. Returns StageResult.
  * ModelEvaluatorAgent — capability(TaskAcceptance.of(EVALUATE_MODEL).maxIterationsPerTask(2)).
    Prompt from prompts/model-evaluator.md. Returns StageResult with a
    populated MetricBundle.

- 1 Workflow PipelineWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> gateStep ->
  dispatchStep -> recordStep -> driftCheckStep -> decideStep
  -> [back to checkHaltStep or to promoteStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120) (covers any specialist call), decideStep ofSeconds(45),
    promoteStep ofSeconds(60). defaultStepRecovery(maxRetries(2).failoverTo
    (PipelineWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
  haltedStep (emits RunHaltedOperator on PipelineRunEntity).
  gateStep runs EvalGate.check(evalLedger, StageDispatch); on GATE_FAILED
  records a StageGateFailed entry via PipelineRunEntity.recordGateFailure(stageName, reason)
  and loops back to proposeStep.
  dispatchStep uses switch on StageDispatch.specialist to call the
  matching specialist agent via forAutonomousAgent(...).runSingleTask(...)
  then forTask(runId).result(...).
  recordStep calls PipelineRunEntity.recordStage(entry).
  driftCheckStep calls DriftEvaluator.check(entry, baselineMetrics) and,
  when drift detected, calls PipelineRunEntity.raiseDriftAlert(alert).
  decideStep calls forAutonomousAgent(PlannerAgent.class, DECIDE_STAGE);
  on Continue or Replan loops; on Promote transitions to composeReportStep
  -> promoteStep; on Fail transitions to failStep.

- 1 EventSourcedEntity PipelineRunEntity holding PipelineRun state. emptyState()
  returns PipelineRun.initial("", "", null) with no commandContext() reference.
  Commands: createRun, recordPlan, recordDispatch, recordGateFailure, recordStage,
  reviseLedger, completeRun, failRun, haltAutomatic, haltOperator, timeoutFail,
  raiseDriftAlert, getRun. Events as listed in SPEC §5.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity RunQueue with command enqueueRun(runId, datasetRef,
  objective, requestedBy) emitting RunSubmitted.

- 1 View PipelineRunView with row type RunRow (mirror of PipelineRun minus
  heavy ledger payloads — truncate to last 3 eval entries plus counts; the UI
  fetches the full run by id on click). Table updater consumes PipelineRunEntity
  events. ONE query getAllRuns SELECT * AS runs FROM pipeline_run_view. No WHERE
  status filter — caller filters client-side (Lesson 2).

- 1 Consumer RunRequestConsumer subscribed to RunQueue events; on RunSubmitted
  starts a PipelineWorkflow with runId as the workflow id.

- 3 TimedActions:
  * RunSimulator — every 90s, reads next line from
    src/main/resources/sample-events/run-prompts.jsonl and calls
    RunQueue.enqueueRun.
  * StaleRunMonitor — every 60s, queries PipelineRunView.getAllRuns, filters
    EXECUTING runs whose createdAt is older than 10 minutes, calls
    PipelineRunEntity.timeoutFail; PipelineWorkflow polls PipelineRunEntity.getRun
    in its decideStep and exits when status == STUCK.
  * DriftWatchMonitor — every 5min, queries PipelineRunView for COMPLETED runs
    updated in the last hour, loads each run's MetricBundle, compares against
    baselines loaded from src/main/resources/sample-data/baselines/metrics-baseline.json,
    calls PipelineRunEntity.raiseDriftAlert when any metric delta exceeds threshold.

- 2 HttpEndpoints:
  * PipelineEndpoint at /api with POST /runs, GET /runs (filters client-side
    from getAllRuns), GET /runs/{id}, GET /runs/sse, GET /alerts,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN_PIPELINE
  (resultConformsTo PipelineLedger), DECIDE_STAGE (NextStage), COMPOSE_REPORT (ModelReport).
- SpecialistTasks.java declaring four Task<R> constants: PROFILE_DATA,
  ENGINEER_FEATURES, TRAIN_MODEL, EVALUATE_MODEL (all resultConformsTo StageResult).
- Domain records as listed in SPEC §5, plus a NextStage sealed interface with
  permits Continue, Replan, Promote, Fail (each carrying its own payload —
  Continue with StageDispatch, Replan with revisedLedger, Promote with
  ModelReport stub, Fail with failureReason).
- application/EvalGate.java — deterministic gate check. Gate passes when the
  evaluation ledger contains at least one MODEL_EVALUATOR entry with
  metrics.accuracy >= 0.80 AND metrics.fairnessDelta <= 0.05 AND
  gateOutcome == PASSED. Gate fails otherwise; returns a GateResult{passed, reason}.
- application/DriftEvaluator.java — deterministic drift check. Compares each
  MetricBundle field against the matching baseline value. Raises an alert when
  |observed - baseline| > 0.10 for accuracy, f1, or auc, or when fairnessDelta
  > 0.05. Returns List<DriftAlert>; empty list means no drift detected.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9762 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/run-prompts.jsonl with 8 canned pipeline
  requests spanning classification, regression, and anomaly detection objectives.
- src/main/resources/sample-data/datasets/ — 4 short CSV-schema fixture files
  (credit-risk-schema.json, churn-schema.json, fraud-schema.json,
  demand-forecast-schema.json) used by DataProfilerAgent.
- src/main/resources/sample-data/baselines/metrics-baseline.json — baseline
  MetricBundle per dataset, used by DriftWatchMonitor.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the project-root files for the metadata endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, E1) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data, decisions,
  failure, oversight, operations.external_tool_calls, and compliance.capabilities;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/planner.md, prompts/data-profiler.md, prompts/feature-engineer.md,
  prompts/model-trainer.md, prompts/model-evaluator.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: ML Engineering Pipeline",
  one-line pitch, prerequisites (including the integration form's host-software
  requirement: None), generate-the-system, what-you-get, customise-before-
  generating, what-gets-validated, license. NO Configuration section. NO
  governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator halt/resume control + live list with
  status pills and expand-on-click for ledgers, drift alerts, and report).
  Browser title exactly: <title>Akka Sample: ML Engineering Pipeline</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  and the Task<R> id.
- Per-agent mock-response shapes for THIS blueprint:
    planner.json — three lists keyed by task id:
      "PLAN_PIPELINE" → 4–6 PipelineLedger entries covering classification and
      regression pipelines with 4–6 ordered stages.
      "DECIDE_STAGE" → 4–6 NextStage entries covering Continue across all four
      specialists, one Replan, one Promote, one Fail.
      "COMPOSE_REPORT" → 4–6 ModelReport entries with 60–120 word summaries
      and MetricBundle values in realistic ranges.
    data-profiler.json — 6 StageResult entries with schema stats as content.
    feature-engineer.json — 6 StageResult entries with FeaturePlan JSON content.
    model-trainer.json — 6 StageResult entries with training metadata content.
    model-evaluator.json — 6 StageResult entries; at least two with
      MetricBundle accuracy >= 0.80 (gate passes), two with accuracy < 0.80
      (gate fails), to exercise J2. One entry includes a MetricBundle with
      fairnessDelta = 0.12 to exercise the drift watch in J4.
- MockModelProvider.seedFor(runId) makes selection deterministic per run id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, dispatchStep, decideStep,
  promoteStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the PipelineRun entity state.
- Lesson 7: AutonomousAgent requires companion PlannerTasks.java and
  SpecialistTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: HTTP port 9762 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND
  themeVariables (state-diagram label colour, edge-label foreignObject
  overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
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
