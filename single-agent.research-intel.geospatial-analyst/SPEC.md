# SPEC — earth-engine-geospatial

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** EarthEngineGeospatialAnalyst.
**One-line pitch:** A user submits a bounding region and a list of geospatial indicators to analyse; one AI agent receives the indicator dataset as a task attachment (never as inline prompt text) and returns a structured `AnalysisReport` — findings, detected anomalies, and recommended follow-up observations per requested indicator.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intelligence domain. One `GeospatialAnalystAgent` (AutonomousAgent) carries the entire interpretation decision; surrounding components prepare the dataset, validate the spatial query, and audit the output. Three governance mechanisms sit around the agent:

- A **coordinate-bounds sanitizer** runs inside a Consumer before the agent call — queries whose bounding box exceeds the maximum permitted area (hard-coded as 500 000 km²) are rejected before any dataset is assembled.
- A **before-agent-response guardrail** validates the agent's report on every turn: well-formed JSON, every finding references a real indicator id, every anomaly confidence value is in the range [0.0, 1.0], and the report covers every requested indicator.
- An **on-decision-eval** runs immediately after each `ReportRecorded` event, scoring the report on evidence quality (does every finding cite a dataset layer and a pixel-value or statistic? are anomaly descriptions specific? are recommended observations actionable?).

The blueprint shows that the single-agent pattern is not "ungoverned": three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a **Query label** (free text), a **Bounding region** (latitude/longitude pair for south-west and north-east corners), and an optional **date range** (ISO-8601 start and end date).
2. The user picks an **indicator set** from a dropdown (Vegetation, Surface Temperature, Flood Extent) or pastes a custom list of indicators.
3. The user clicks **Submit query**. The UI POSTs to `/api/analyses` and receives an `analysisId`.
4. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `DATASET_READY` — the fetched dataset summary is visible in the card detail (layer count, pixel resolution, missing-data percentage).
5. Within ~10–30 s, the workflow's `analysisStep` completes. The card transitions to `ANALYSING` then `REPORT_RECORDED`. The report appears: a top-level status badge (NOMINAL / ANOMALY_DETECTED / INSUFFICIENT_DATA), a short summary paragraph, and a per-finding table (indicator id, severity, layer, statistic, anomaly flag, recommendation).
6. Within ~1 s of the report, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the evidence is solid.
7. The user can submit another query; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/analyses/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AnalysisEntity`, `AnalysisView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AnalysisEntity` | `EventSourcedEntity` | Per-analysis lifecycle: submitted → dataset_ready → analysing → report_recorded → evaluated. Source of truth. | `AnalysisEndpoint`, `DatasetFetcher`, `AnalysisWorkflow` | `AnalysisView` |
| `DatasetFetcher` | `Consumer` | Subscribes to `QuerySubmitted` events; validates bounding box area; assembles indicator snapshots; calls `AnalysisEntity.attachDataset`. | `AnalysisEntity` events | `AnalysisEntity` |
| `AnalysisWorkflow` | `Workflow` | One workflow per analysis. Steps: `awaitDatasetStep` → `analysisStep` → `evalStep`. | started by `DatasetFetcher` once dataset event lands | `GeospatialAnalystAgent`, `AnalysisEntity` |
| `GeospatialAnalystAgent` | `AutonomousAgent` | The one decision-making LLM. Receives indicator definitions in the task instructions and the assembled dataset as a task attachment; returns `AnalysisReport`. | invoked by `AnalysisWorkflow` | returns report |
| `AnalysisView` | `View` | Read model: one row per analysis for the UI. | `AnalysisEntity` events | `AnalysisEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Indicator(String indicatorId, String name, String datasetLayer, String unit) {}

record BoundingBox(double swLat, double swLon, double neLat, double neLon) {}

record AnalysisQuery(
    String analysisId,
    String queryLabel,
    BoundingBox bounds,
    String startDate,   // ISO-8601 date, e.g. "2024-01-01"
    String endDate,     // ISO-8601 date, e.g. "2024-12-31"
    List<Indicator> indicators,
    String submittedBy,
    Instant submittedAt
) {}

record LayerStats(
    String indicatorId,
    String datasetLayer,
    double meanValue,
    double minValue,
    double maxValue,
    double missingDataPct,   // 0.0–100.0
    String unit
) {}

record DatasetSnapshot(
    List<LayerStats> layers,
    int pixelResolutionMeters,
    long totalPixels
) {}

record Finding(
    String indicatorId,
    FindingSeverity severity,
    String datasetLayer,
    String statistic,        // e.g. "mean NDVI = 0.23 (threshold 0.35)"
    boolean anomalyDetected,
    double anomalyConfidence,  // 0.0–1.0
    String recommendation
) {}
enum FindingSeverity { INFO, WATCH, ALERT, CRITICAL }

record AnalysisReport(
    ReportStatus status,
    String summary,
    List<Finding> findings,
    Instant decidedAt
) {}
enum ReportStatus { NOMINAL, ANOMALY_DETECTED, INSUFFICIENT_DATA }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Analysis(
    String analysisId,
    Optional<AnalysisQuery> query,
    Optional<DatasetSnapshot> dataset,
    Optional<AnalysisReport> report,
    Optional<EvalResult> eval,
    AnalysisStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AnalysisStatus {
    SUBMITTED, DATASET_READY, ANALYSING, REPORT_RECORDED, EVALUATED, FAILED
}
```

Events on `AnalysisEntity`: `QuerySubmitted`, `DatasetReady`, `AnalysisStarted`, `ReportRecorded`, `EvaluationScored`, `AnalysisFailed`.

Every nullable lifecycle field on the `Analysis` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/analyses` — body `{ queryLabel, bounds: {swLat, swLon, neLat, neLon}, startDate, endDate, indicators: [Indicator], submittedBy }` → `{ analysisId }`.
- `GET /api/analyses` — list all analyses, newest-first.
- `GET /api/analyses/{id}` — one analysis.
- `GET /api/analyses/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Earth Engine Geospatial Analyst</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted analyses (status pill + report status badge + age) and a right pane with the selected analysis's detail — submitted indicators list, dataset snapshot preview, report summary, finding table, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Coordinate-bounds sanitizer** (`sanitizer`, applied inside `DatasetFetcher` Consumer): rejects any bounding box whose computed area exceeds 500 000 km². Validates that south-west coordinates are numerically less than north-east coordinates. Records the computed area on the `DatasetSnapshot`. An oversized or invalid query transitions the entity to `FAILED` before any dataset assembly begins.
- **G1 — before-agent-response guardrail**: runs on every turn of `GeospatialAnalystAgent`. Asserts the candidate response is well-formed `AnalysisReport` JSON, every `findings[].indicatorId` matches a submitted indicator, every `anomalyConfidence` is in `[0.0, 1.0]`, and the report covers every requested indicator. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `ReportRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that every finding cites a `datasetLayer` and a non-empty `statistic`, that recommendations are actionable verbs, and that anomaly-confidence values are not uniformly zero when `anomalyDetected` is true. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `GeospatialAnalystAgent` → `prompts/geospatial-analyst.md`. The single decision-making LLM. System prompt instructs it to read the attached dataset, walk every indicator, and return one `Finding` per indicator citing a layer statistic as evidence.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a Vegetation indicator query over a valid bounding box; within 30 s the report appears with one finding per indicator and an eval score chip.
2. **J2** — The agent's first response is intentionally malformed (mock LLM path) — the guardrail rejects it; the second iteration produces a well-formed report; the UI never displays the malformed response.
3. **J3** — A report whose findings all carry empty `statistic` strings receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A query whose bounding box area exceeds 500 000 km² is rejected by `DatasetFetcher`; the entity transitions to `FAILED` before any LLM call is made; the UI card shows the failure reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named earth-engine-geospatial demonstrating the single-agent × research-intel
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-geospatial-analyst. Java package
io.akka.samples.earthenginegeospatial. Akka 3.6.0. HTTP port 9104.

Components to wire (exactly):

- 1 AutonomousAgent GeospatialAnalystAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/geospatial-analyst.md>) and
  .capability(TaskAcceptance.of(ANALYSE_REGION).maxIterationsPerTask(3)). The task receives
  indicator definitions as its instruction text and the assembled dataset as a task ATTACHMENT
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: AnalysisReport{status: ReportStatus, summary: String, findings: List<Finding>,
  decidedAt: Instant}. The agent is configured with a before-agent-response guardrail (see G1
  in eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow AnalysisWorkflow per analysisId with three steps:
  * awaitDatasetStep — polls AnalysisEntity.getAnalysis every 1s; on
    analysis.dataset().isPresent() advances to analysisStep.
    WorkflowSettings.stepTimeout 15s.
  * analysisStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    GeospatialAnalystAgent.class, "analyst-" + analysisId).runSingleTask(
      TaskDef.instructions(formatIndicators(analysis.query.indicators))
        .attachment("dataset.json", serializeSnapshot(analysis.dataset).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANALYSE_REGION) to fetch the report.
    On success calls AnalysisEntity.recordReport(report). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(AnalysisWorkflow::error).
  * evalStep — runs a deterministic rule-based EvaluationScorer (NOT an LLM call) over the
    recorded report: checks that every finding has non-empty statistic and datasetLayer, that
    recommendations are actionable, and that anomalyConfidence is non-zero when anomalyDetected
    is true. Emits EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity AnalysisEntity (one per analysisId). State Analysis{analysisId: String,
  query: Optional<AnalysisQuery>, dataset: Optional<DatasetSnapshot>,
  report: Optional<AnalysisReport>, eval: Optional<EvalResult>, status: AnalysisStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. AnalysisStatus enum: SUBMITTED,
  DATASET_READY, ANALYSING, REPORT_RECORDED, EVALUATED, FAILED. Events:
  QuerySubmitted{query}, DatasetReady{dataset}, AnalysisStarted{}, ReportRecorded{report},
  EvaluationScored{eval}, AnalysisFailed{reason}. Commands: submit, attachDataset,
  markAnalysing, recordReport, recordEvaluation, fail, getAnalysis. emptyState() returns
  Analysis.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer DatasetFetcher subscribed to AnalysisEntity events; on QuerySubmitted: first
  validates the bounding box (computes area using the haversine formula; rejects if area >
  500 000 km² or if swLat >= neLat or swLon >= neLon by calling AnalysisEntity.fail(reason));
  on valid bounds, assembles a DatasetSnapshot from the seeded indicator JSONL, one LayerStats
  entry per requested indicator. Calls AnalysisEntity.attachDataset(snapshot). After
  attachDataset lands, starts an AnalysisWorkflow with id = "analysis-" + analysisId.

- 1 View AnalysisView with row type AnalysisRow (mirrors Analysis minus query.submittedBy
  for privacy). Table updater consumes AnalysisEntity events. ONE query getAllAnalyses:
  SELECT * AS analyses FROM analysis_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /analyses (body
    {queryLabel, bounds: {swLat, swLon, neLat, neLon}, startDate, endDate,
    indicators: [{indicatorId, name, datasetLayer, unit}], submittedBy};
    mints analysisId; calls AnalysisEntity.submit; returns {analysisId}), GET /analyses
    (list from getAllAnalyses, sorted newest-first), GET /analyses/{id} (one row), GET
    /analyses/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AnalysisTasks.java declaring one Task<R> constant: ANALYSE_REGION = Task.name("Analyse
  region").description("Read the attached dataset snapshot and produce an AnalysisReport
  per indicator").resultConformsTo(AnalysisReport.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Indicator, BoundingBox, AnalysisQuery, LayerStats, DatasetSnapshot, Finding,
  FindingSeverity, AnalysisReport, ReportStatus, EvalResult, Analysis, AnalysisStatus.

- ReportGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AnalysisReport from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- EvaluationScorer.java — pure deterministic logic (no LLM). Inputs: AnalysisReport and the
  list of submitted Indicators. Outputs: EvalResult. Scoring rubric documented in Javadoc.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9104 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. GeospatialAnalystAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/indicators.jsonl with 3 seeded indicator sets:
  a 5-indicator Vegetation catalogue (NDVI, EVI, LAI, chlorophyll index, greenness),
  a 4-indicator Surface Temperature catalogue (LST day, LST night, urban heat island proxy,
  albedo), and a 4-indicator Flood Extent catalogue (SAR backscatter, water mask, inundation
  depth proxy, soil moisture).

- src/main/resources/sample-events/seed-datasets.jsonl with 3 paired example DatasetSnapshot
  objects: one per indicator set, with realistic meanValue / minValue / maxValue ranges, ~2%
  missingDataPct, pixelResolutionMeters = 30, and enough variation to produce non-trivial
  agent findings.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.location-fine-grained = true,
  decisions.authority_level = recommend-only (the agent's report is advisory, not enforced),
  oversight.human_in_loop = true (a researcher reads the report before acting on it),
  failure.failure_modes including "hallucinated-anomaly", "missed-indicator",
  "false-alarm-fatigue", "confidence-miscalibration"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/geospatial-analyst.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Earth Engine Geospatial Analyst",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of analysis cards; right = selected-analysis detail with submitted indicators,
  dataset snapshot preview, report summary, finding table, and eval-score chip).
  Browser title exactly: <title>Akka Sample: Earth Engine Geospatial Analyst</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; gone when
        the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(analysisId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    analyse-region.json — 8 AnalysisReport entries covering the three ReportStatus values.
      Each entry has a summary paragraph and a findings array with one Finding per indicator
      in the matched set. Each Finding has a non-empty datasetLayer, a non-empty statistic
      (e.g. "mean NDVI = 0.23"), and an actionable recommendation. Severities and
      anomalyDetected values vary realistically. Plus 2 deliberately MALFORMED entries (one
      with a finding whose indicatorId is not in the submitted list; one with anomalyConfidence
      outside [0.0, 1.0]) — the guardrail blocks both. The mock selects a malformed entry on
      the FIRST iteration of every 3rd analysis (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(analysisId) helper makes per-analysis selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeospatialAnalystAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion AnalysisTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (analysisStep 60s, awaitDatasetStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Analysis row record is Optional<T>.
- Lesson 7: AnalysisTasks.java with ANALYSE_REGION = Task.name(...).description(...)
  .resultConformsTo(AnalysisReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9104 declared explicitly in application.conf's
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
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (GeospatialAnalystAgent). The on-decision eval is rule-based (EvaluationScorer.java) and
  does NOT make an LLM call.
- The dataset is passed as a Task ATTACHMENT, never inlined into the agent's instructions.
  The generated analysisStep MUST use TaskDef.attachment(...) not string interpolation.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
