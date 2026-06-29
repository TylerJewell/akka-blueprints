# SPEC — time-series-forecasting (java)

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** TSForecast.
**One-line pitch:** A user uploads a labeled tabular time series and a forecast configuration; one AI agent reads the series (passed as a task attachment, never as inline prompt text) and returns a structured `ForecastResult` — point forecasts, confidence intervals, and a per-horizon quality score for each requested horizon step.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `ForecastingAgent` (AutonomousAgent) carries the entire forecast decision; the surrounding components only prepare its input, validate the series, and audit its output. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates the agent's forecast result on every turn: well-formed JSON, every horizon step has a non-null point value, every confidence interval is logically ordered (`lower ≤ point ≤ upper`), and the horizon count matches the requested count. A malformed result triggers a retry inside the same task.

A periodic drift-fairness watch evaluator runs on each completed forecast:

- An **eval-periodic drift-fairness-watch** runs immediately after each `ForecastCompleted` event, scoring the forecast for distributional drift relative to the series' own historical baseline. This is a deterministic statistical check (no LLM call) that computes mean absolute percentage error (MAPE) against the series' last-known actuals window and flags results where MAPE exceeds the configured threshold.

The blueprint shows that a single-agent forecasting pattern does not mean "unmonitored" — a guardrail on the output format and a drift watch on every result together give deployers early warning of model degradation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **series** from a dropdown (Revenue, Inventory Demand, Credit Loss) or pastes a CSV-formatted tabular series (date, value columns).
2. The user sets a **forecast horizon** (7 / 14 / 30 days) and a **confidence level** (80% / 95%).
3. The user clicks **Submit for forecast**. The UI POSTs to `/api/forecasts` and receives a `forecastId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `VALIDATED` — the series quality summary is visible in the card detail (row count, date range, missing-value count, stationarity flag).
5. Within ~10–30 s, the workflow's `forecastStep` completes. The card transitions to `FORECASTING` then `FORECAST_COMPLETED`. The result appears: a per-horizon table (date, point forecast, lower bound, upper bound), a horizon-quality-score column (1–5 per step), and a short summary paragraph.
6. Within ~1 s of the forecast, the `driftEvalStep` finishes. The card shows a **drift score** chip (STABLE / WATCH / ALERT) and a MAPE value.
7. The user can submit another series; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ForecastEndpoint` | `HttpEndpoint` | `/api/forecasts/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ForecastEntity`, `ForecastView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ForecastEntity` | `EventSourcedEntity` | Per-forecast lifecycle: submitted → validated → forecasting → completed → evaluated. Source of truth. | `ForecastEndpoint`, `SeriesValidator`, `ForecastWorkflow` | `ForecastView` |
| `SeriesValidator` | `Consumer` | Subscribes to `SeriesSubmitted` events; checks data quality and stationarity; calls `ForecastEntity.attachValidated`. | `ForecastEntity` events | `ForecastEntity` |
| `ForecastWorkflow` | `Workflow` | One workflow per forecast. Steps: `awaitValidatedStep` → `forecastStep` → `driftEvalStep`. | started by `SeriesValidator` once validated event lands | `ForecastingAgent`, `ForecastEntity` |
| `ForecastingAgent` | `AutonomousAgent` | The one decision-making LLM. Receives forecast configuration in the task definition and the validated series as a task attachment; returns `ForecastResult`. | invoked by `ForecastWorkflow` | returns result |
| `ForecastView` | `View` | Read model: one row per forecast for the UI. | `ForecastEntity` events | `ForecastEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SeriesConfig(
    int horizonDays,
    double confidenceLevel,      // 0.80 or 0.95
    String granularity           // "daily" | "weekly"
) {}

record SeriesPoint(
    LocalDate date,
    double value
) {}

record SeriesSubmission(
    String forecastId,
    String seriesName,
    List<SeriesPoint> historicalData,
    SeriesConfig config,
    String submittedBy,
    Instant submittedAt
) {}

record SeriesQuality(
    int rowCount,
    LocalDate rangeStart,
    LocalDate rangeEnd,
    int missingValueCount,
    boolean isStationary,
    String qualitySummary
) {}

record HorizonStep(
    LocalDate forecastDate,
    double pointForecast,
    double lowerBound,
    double upperBound,
    int stepQualityScore          // 1..5; decreases for longer horizons
) {}
// Invariant: lowerBound <= pointForecast <= upperBound

record ForecastResult(
    List<HorizonStep> steps,
    String summary,
    Instant completedAt
) {}

record DriftEval(
    DriftStatus driftStatus,
    double mape,                  // mean absolute percentage error vs. actuals window
    String rationale,
    Instant evaluatedAt
) {}
enum DriftStatus { STABLE, WATCH, ALERT }

record Forecast(
    String forecastId,
    Optional<SeriesSubmission> submission,
    Optional<SeriesQuality> quality,
    Optional<ForecastResult> result,
    Optional<DriftEval> driftEval,
    ForecastStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ForecastStatus {
    SUBMITTED, VALIDATED, FORECASTING, FORECAST_COMPLETED, DRIFT_EVALUATED, FAILED
}
```

Events on `ForecastEntity`: `SeriesSubmitted`, `SeriesValidated`, `ForecastStarted`, `ForecastCompleted`, `DriftEvaluated`, `ForecastFailed`.

Every nullable lifecycle field on the `Forecast` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/forecasts` — body `{ seriesName, historicalData: [SeriesPoint], config: SeriesConfig, submittedBy }` → `{ forecastId }`.
- `GET /api/forecasts` — list all forecasts, newest-first.
- `GET /api/forecasts/{id}` — one forecast.
- `GET /api/forecasts/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: TSForecast</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted forecasts (status pill + drift badge + age) and a right pane with the selected forecast's detail — series quality summary, forecast table with per-horizon bounds, summary paragraph, and drift-eval chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ForecastingAgent`. Asserts the candidate response is well-formed `ForecastResult` JSON, every `steps[].pointForecast` is non-null, every confidence interval satisfies `lowerBound ≤ pointForecast ≤ upperBound`, and the `steps` list has exactly `config.horizonDays / granularity-factor` entries. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — drift-fairness-watch eval** (`eval-periodic`, `drift-fairness-watch`): runs immediately after `ForecastCompleted` lands, as `driftEvalStep` inside the workflow. A deterministic statistical scorer (no LLM call) computes MAPE over the last `min(30, rowCount)` actuals. If MAPE < 10% → `STABLE`; 10–25% → `WATCH`; > 25% → `ALERT`. Emits `DriftEvaluated` with status, MAPE value, and a one-line rationale.

## 9. Agent prompts

- `ForecastingAgent` → `prompts/forecasting-agent.md`. The single decision-making LLM. System prompt instructs it to analyse the attached series, identify trend and seasonality, and return a `ForecastResult` with one `HorizonStep` per requested horizon day.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the Revenue seed series with a 14-day horizon; within 30 s the forecast appears with 14 horizon steps, each with non-empty point and bounds, and a drift-eval chip.
2. **J2** — The agent's first response on a forecast is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed result; the UI never displays the malformed response.
3. **J3** — A forecast whose MAPE against the actuals window exceeds 25% receives a drift status of `ALERT`; the UI card border highlights red.
4. **J4** — A series submitted with mixed-currency values (non-numeric rows) triggers a `FAILED` status from `SeriesValidator`; the entity records `ForecastFailed` with a clear reason; the UI shows the failed card without crashing.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named tsforecast demonstrating the single-agent × finance-analysis cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-java-ts-forecast. Java package
io.akka.samples.timeseriesforecastingjava. Akka 3.6.0. HTTP port 9643.

Components to wire (exactly):

- 1 AutonomousAgent ForecastingAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/forecasting-agent.md>) and
  .capability(TaskAcceptance.of(FORECAST_SERIES).maxIterationsPerTask(3)). The task receives
  forecast configuration (horizonDays, confidenceLevel, granularity) as its instruction text
  and the validated CSV series as a task ATTACHMENT named "series.csv"
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: ForecastResult{steps: List<HorizonStep>, summary: String, completedAt:
  Instant}. The agent is configured with a before-agent-response guardrail (see G1 in
  eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries the response within its 3-iteration budget.

- 1 Workflow ForecastWorkflow per forecastId with three steps:
  * awaitValidatedStep — polls ForecastEntity.getForecast every 1s; on
    forecast.quality().isPresent() advances to forecastStep. WorkflowSettings.stepTimeout
    15s (validator is in-process and fast).
  * forecastStep — emits ForecastStarted, then calls componentClient.forAutonomousAgent(
    ForecastingAgent.class, "forecaster-" + forecastId).runSingleTask(
      TaskDef.instructions(formatConfig(forecast.submission.config))
        .attachment("series.csv", toCsvBytes(forecast.quality))
    ) — returns a taskId, then forTask(taskId).result(FORECAST_SERIES) to fetch the result.
    On success calls ForecastEntity.recordForecast(result). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(ForecastWorkflow::error).
  * driftEvalStep — runs a deterministic statistical DriftEvaluator (NOT an LLM call) over the
    recorded ForecastResult: computes MAPE against the last actuals window from
    submission.historicalData. STABLE if MAPE < 0.10; WATCH if 0.10–0.25; ALERT if > 0.25.
    Emits DriftEvaluated{driftStatus, mape, rationale, evaluatedAt}. WorkflowSettings
    .stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ForecastEntity (one per forecastId). State Forecast{forecastId:
  String, submission: Optional<SeriesSubmission>, quality: Optional<SeriesQuality>,
  result: Optional<ForecastResult>, driftEval: Optional<DriftEval>, status: ForecastStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. ForecastStatus enum: SUBMITTED,
  VALIDATED, FORECASTING, FORECAST_COMPLETED, DRIFT_EVALUATED, FAILED. Events:
  SeriesSubmitted{submission}, SeriesValidated{quality}, ForecastStarted{},
  ForecastCompleted{result}, DriftEvaluated{driftEval}, ForecastFailed{reason}. Commands:
  submit, attachValidated, markForecasting, recordForecast, recordDriftEval, fail,
  getForecast. emptyState() returns Forecast.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer SeriesValidator subscribed to ForecastEntity events; on SeriesSubmitted runs
  data-quality checks (row count ≥ 14, no non-numeric value column, no gap > 7 days,
  date column parseable as ISO-8601 LocalDate), computes stationarity via an ADF
  approximation (rolling mean variance ratio), builds SeriesQuality, then calls
  ForecastEntity.attachValidated(quality). After attachValidated lands, the same Consumer
  starts a ForecastWorkflow with id = "forecast-" + forecastId. If any check fails,
  Consumer calls ForecastEntity.fail(reason) instead.

- 1 View ForecastView with row type ForecastRow (mirrors Forecast minus
  submission.historicalData — the audit log keeps the raw series; the view holds quality
  summary for the UI). Table updater consumes ForecastEntity events. ONE query
  getAllForecasts: SELECT * AS forecasts FROM forecast_view. No WHERE status filter — Akka
  cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ForecastEndpoint at /api with POST /forecasts (body
    {seriesName, historicalData: [{date, value}], config: {horizonDays, confidenceLevel,
    granularity}, submittedBy}; mints forecastId; calls ForecastEntity.submit; returns
    {forecastId}), GET /forecasts (list from getAllForecasts, sorted newest-first), GET
    /forecasts/{id} (one row), GET /forecasts/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ForecastTasks.java declaring one Task<R> constant: FORECAST_SERIES = Task.name("Forecast
  series").description("Read the attached historical series and produce a ForecastResult
  with one HorizonStep per requested horizon day").resultConformsTo(ForecastResult.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records SeriesConfig, SeriesPoint, SeriesSubmission, SeriesQuality, HorizonStep,
  ForecastResult, DriftEval, DriftStatus, Forecast, ForecastStatus.

- ForecastGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ForecastResult from the LLM response, runs the three checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- DriftEvaluator.java — pure deterministic logic (no LLM). Inputs: ForecastResult and the
  historical actuals from SeriesSubmission. Outputs: DriftEval. Scoring rubric documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9643 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ForecastingAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/series-data.jsonl with 3 seeded time series:
  a 90-day daily revenue series (in USD), a 180-day weekly inventory-demand series (units),
  and a 365-day monthly credit-loss series (bps). Each contains realistic seasonal patterns
  and 1–2 anomaly points.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.financial_time_series = true,
  decisions.authority_level = recommend-only (the agent's forecast is advisory input to
  downstream models and human analysts), oversight.human_in_loop = true, failure.failure_modes
  including "horizon-overconfidence", "distributional-shift-undetected",
  "missing-seasonality", "interval-inversion"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/forecasting-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Time-Series Forecasting (Java)",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of forecast cards; right = selected-forecast detail with series quality summary,
  forecast table with per-step bounds, summary paragraph, and drift-eval chip).
  Browser title exactly: <title>Akka Sample: TSForecast</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(forecastId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    forecast-series.json — 8 ForecastResult entries covering STABLE, WATCH, and ALERT
      drift outcomes. Each entry has a summary paragraph and a `steps` array with one
      HorizonStep per day in the matched horizon (7, 14, or 30 days). Each HorizonStep
      has a non-null pointForecast, a lowerBound < pointForecast, and an upperBound >
      pointForecast, and a stepQualityScore that decreases toward the end of the horizon.
      Plus 2 deliberately MALFORMED entries (one with lowerBound > pointForecast; one with
      steps count mismatching the requested horizon) — the guardrail blocks both, exercising
      the retry path. The mock selects a malformed entry on the FIRST iteration of every 3rd
      forecast (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(forecastId) helper makes per-forecast selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ForecastingAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ForecastTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (forecastStep
  60s, awaitValidatedStep 15s, driftEvalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Forecast row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ForecastTasks.java with FORECAST_SERIES = Task.name(...).description(...)
  .resultConformsTo(ForecastResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9643 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ForecastingAgent). The
  drift evaluator is rule-based (DriftEvaluator.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The series is passed as a Task ATTACHMENT named "series.csv", never inlined into the
  agent's instructions. Verify the generated forecastStep uses TaskDef.attachment(...)
  and not string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference for how the hook is registered.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
