# SPEC — youtube-analyst

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** YouTubeAnalyst.
**One-line pitch:** A user requests a YouTube channel analysis; one AI agent reads the fetched channel metrics and video performance data (passed as task context, never as inline prompt text) and returns a structured performance report — top videos, engagement trends, audience signals, and growth recommendations.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `ChannelAnalystAgent` (AutonomousAgent) carries the entire analytical decision; the surrounding components only prepare its input and audit its output. Two governance mechanisms are wired around the agent:

- A **before-agent-response guardrail** validates the agent's report on every turn: well-formed JSON, every cited video references a real `videoId` from the fetched data set, every trend claim references a named metric, and the recommendations list is non-empty. A malformed report triggers a retry inside the same task.
- An **on-decision-eval** runs immediately after each `ReportRecorded` event, scoring the report on analytical completeness (are trend claims backed by numeric evidence? do recommendations reference specific metrics? is the engagement-rate calculation present and non-zero?).

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **channel** from a dropdown (three seeded fixtures — a consumer-electronics brand, a developer-tools creator, a lifestyle vlogger) or enters a custom channel handle.
2. The user picks an **analysis period** (last 28 days / last 90 days / all-time) and an **focus area** (Growth, Engagement, Audience, or All).
3. The user clicks **Run analysis**. The UI POSTs to `/api/analyses` and receives an `analysisId`.
4. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `METRICS_FETCHED` — the fetched video count, subscriber delta, and top-metric values are visible in the card header.
5. Within ~10–30 s, the workflow's `analyseStep` completes. The card transitions to `ANALYSING` then `REPORT_RECORDED`. The report appears: a performance tier badge (GROWING / STEADY / DECLINING), a narrative summary paragraph, a top-videos table (title, views, engagement rate, trend arrow), and a recommendation list.
6. Within ~1 s of the report, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the report's trend claims are properly grounded in the numbers.
7. The user can request analysis on another channel; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/analyses/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ChannelEntity`, `AnalysisView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ChannelEntity` | `EventSourcedEntity` | Per-analysis lifecycle: requested → metrics-fetched → analysing → report → eval. Source of truth. | `AnalysisEndpoint`, `MetricsFetcher`, `AnalysisWorkflow` | `AnalysisView` |
| `MetricsFetcher` | `Consumer` | Subscribes to `AnalysisRequested` events; retrieves channel metrics and video data from seeded fixtures; calls `ChannelEntity.attachMetrics`. | `ChannelEntity` events | `ChannelEntity` |
| `AnalysisWorkflow` | `Workflow` | One workflow per analysis. Steps: `awaitMetricsStep` → `analyseStep` → `evalStep`. | started by `MetricsFetcher` once metrics event lands | `ChannelAnalystAgent`, `ChannelEntity` |
| `ChannelAnalystAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the analysis focus area in the task definition and the channel metrics as a task attachment; returns `ChannelReport`. | invoked by `AnalysisWorkflow` | returns report |
| `AnalysisView` | `View` | Read model: one row per analysis for the UI. | `ChannelEntity` events | `AnalysisEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record VideoMetric(
    String videoId,
    String title,
    long views,
    long likes,
    long comments,
    double engagementRate,   // (likes + comments) / views
    String publishedAt       // ISO-8601 date string
) {}

record ChannelMetrics(
    String channelId,
    String channelHandle,
    long subscriberCount,
    long subscriberDelta,    // change over the requested period
    long totalViews,
    long viewsDelta,
    String period,           // "28d" | "90d" | "all"
    List<VideoMetric> topVideos,
    Instant fetchedAt
) {}

record AnalysisRequest(
    String analysisId,
    String channelHandle,
    String period,
    FocusArea focusArea,
    String requestedBy,
    Instant requestedAt
) {}
enum FocusArea { GROWTH, ENGAGEMENT, AUDIENCE, ALL }

record TopVideo(
    String videoId,
    String title,
    long views,
    double engagementRate,
    TrendDirection trend
) {}
enum TrendDirection { UP, FLAT, DOWN }

record Recommendation(
    String metricRef,   // which metric this recommendation targets
    String action       // actionable verb-phrase
) {}

record ChannelReport(
    PerformanceTier tier,
    String summary,
    List<TopVideo> topVideos,
    List<Recommendation> recommendations,
    Instant reportedAt
) {}
enum PerformanceTier { GROWING, STEADY, DECLINING }

record EvalResult(
    int score,          // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record ChannelAnalysis(
    String analysisId,
    Optional<AnalysisRequest> request,
    Optional<ChannelMetrics> metrics,
    Optional<ChannelReport> report,
    Optional<EvalResult> eval,
    AnalysisStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AnalysisStatus {
    REQUESTED, METRICS_FETCHED, ANALYSING, REPORT_RECORDED, EVALUATED, FAILED
}
```

Events on `ChannelEntity`: `AnalysisRequested`, `MetricsFetched`, `AnalysisStarted`, `ReportRecorded`, `EvaluationScored`, `AnalysisFailed`.

Every nullable lifecycle field on the `ChannelAnalysis` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/analyses` — body `{ channelHandle, period, focusArea, requestedBy }` → `{ analysisId }`.
- `GET /api/analyses` — list all analyses, newest-first.
- `GET /api/analyses/{id}` — one analysis.
- `GET /api/analyses/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: YouTubeAnalyst</title>`.

The App UI tab is a two-column layout: a left rail with the live list of requested analyses (status pill + tier badge + age) and a right pane with the selected analysis's detail — fetched metrics summary, top-videos table, narrative summary, recommendation list, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ChannelAnalystAgent`. Asserts the candidate response is well-formed `ChannelReport` JSON, every `topVideos[].videoId` matches a video in the fetched `ChannelMetrics.topVideos`, every `recommendations[].metricRef` names a field present in the fetched data, and the `recommendations` list is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `ReportRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) checks that the `summary` paragraph references at least one numeric figure from the fetched metrics, that every `TopVideo` has a non-zero `engagementRate`, that `recommendations` list has at least one entry per focus area dimension, and that the `tier` classification is consistent with the `subscriberDelta` sign. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `ChannelAnalystAgent` → `prompts/channel-analyst.md`. The single decision-making LLM. System prompt instructs it to read the attached channel metrics, identify the top-performing videos, characterize engagement and growth trends, and return one structured `ChannelReport`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User requests an analysis of the consumer-electronics fixture; within 30 s the report appears with a tier badge, summary, top-videos table, and recommendation list — plus an eval score chip.
2. **J2** — The agent's first response on an analysis is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed report; the UI never displays the malformed response.
3. **J3** — A report whose `topVideos` have all-zero `engagementRate` values receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A report whose `recommendations` reference a metric not present in the fetched data set is caught by the guardrail before it lands in the entity.
5. **J5** — All three seeded channel fixtures produce distinct `PerformanceTier` classifications over the default 28-day period.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named youtube-analyst demonstrating the single-agent × research-intel cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-youtube-analyst. Java package io.akka.samples.youtubeanalyst.
Akka 3.6.0. HTTP port 9308.

Components to wire (exactly):

- 1 AutonomousAgent ChannelAnalystAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/channel-analyst.md>) and
  .capability(TaskAcceptance.of(ANALYSE_CHANNEL).maxIterationsPerTask(3)). The task receives
  the focus area and period as its instruction text and the channel metrics JSON as a task
  ATTACHMENT (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the
  canonical call). Output: ChannelReport{tier: PerformanceTier, summary: String,
  topVideos: List<TopVideo>, recommendations: List<Recommendation>, reportedAt: Instant}.
  The agent is configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries within its 3-iteration budget.

- 1 Workflow AnalysisWorkflow per analysisId with three steps:
  * awaitMetricsStep — polls ChannelEntity.getAnalysis every 1s; on analysis.metrics().isPresent()
    advances to analyseStep. WorkflowSettings.stepTimeout 15s (fetcher is in-process and fast).
  * analyseStep — emits AnalysisStarted, then calls componentClient.forAutonomousAgent(
    ChannelAnalystAgent.class, "analyst-" + analysisId).runSingleTask(
      TaskDef.instructions(formatFocusInstructions(analysis.request.focusArea, analysis.request.period))
        .attachment("metrics.json", serializeMetrics(analysis.metrics).getBytes())
    ) — returns a taskId, then forTask(taskId).result(ANALYSE_CHANNEL) to fetch the report.
    On success calls ChannelEntity.recordReport(report). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(AnalysisWorkflow::error).
  * evalStep — runs a deterministic rule-based AnalysisEvaluator (NOT an LLM call) over the
    recorded report: checks that summary references at least one numeric figure, that every
    TopVideo has non-zero engagementRate, that recommendations list has at least one entry, and
    that the tier classification is consistent with subscriberDelta sign. Emits
    EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ChannelEntity (one per analysisId). State ChannelAnalysis{analysisId:
  String, request: Optional<AnalysisRequest>, metrics: Optional<ChannelMetrics>,
  report: Optional<ChannelReport>, eval: Optional<EvalResult>, status: AnalysisStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. AnalysisStatus enum: REQUESTED,
  METRICS_FETCHED, ANALYSING, REPORT_RECORDED, EVALUATED, FAILED. Events:
  AnalysisRequested{request}, MetricsFetched{metrics}, AnalysisStarted{},
  ReportRecorded{report}, EvaluationScored{eval}, AnalysisFailed{reason}. Commands:
  requestAnalysis, attachMetrics, markAnalysing, recordReport, recordEvaluation, fail,
  getAnalysis. emptyState() returns ChannelAnalysis.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer MetricsFetcher subscribed to ChannelEntity events; on AnalysisRequested loads the
  seeded channel fixture matching channelHandle from
  src/main/resources/sample-events/channel-fixtures.jsonl, filters to the requested period,
  computes subscriberDelta and viewsDelta, ranks topVideos by engagementRate, builds
  ChannelMetrics, then calls ChannelEntity.attachMetrics(metrics). After attachMetrics lands,
  the same Consumer starts an AnalysisWorkflow with id = "analysis-" + analysisId.

- 1 View AnalysisView with row type AnalysisRow (mirrors ChannelAnalysis). Table updater
  consumes ChannelEntity events. ONE query getAllAnalyses: SELECT * AS analyses FROM
  analysis_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /analyses (body {channelHandle, period, focusArea,
    requestedBy}; mints analysisId; calls ChannelEntity.requestAnalysis; returns {analysisId}),
    GET /analyses (list from getAllAnalyses, sorted newest-first), GET /analyses/{id} (one
    row), GET /analyses/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- AnalysisTasks.java declaring one Task<R> constant: ANALYSE_CHANNEL = Task.name("Analyse
  channel").description("Read the attached channel metrics and produce a ChannelReport with
  tier classification, top-video table, and recommendations").resultConformsTo(ChannelReport.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records AnalysisRequest, ChannelMetrics, VideoMetric, TopVideo, TrendDirection,
  FocusArea, Recommendation, ChannelReport, PerformanceTier, EvalResult, ChannelAnalysis,
  AnalysisStatus.

- ReportGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ChannelReport from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- AnalysisEvaluator.java — pure deterministic logic (no LLM). Inputs: ChannelReport and the
  ChannelMetrics that fed the agent. Outputs: EvalResult. Scoring rubric documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9308 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ChannelAnalystAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/channel-fixtures.jsonl with 3 seeded channel fixtures:
  a consumer-electronics brand (high views, flat engagement), a developer-tools creator
  (moderate views, high engagement), and a lifestyle vlogger (growing subscribers, mixed
  engagement). Each fixture includes 10 videos with realistic view/like/comment counts and
  publish dates spanning the last 180 days.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes reflecting public performance
  metrics (no PII), decisions.authority_level = inform-only (the report surfaces signals;
  a human creator or strategist decides the action), oversight.human_in_loop = true,
  failure.failure_modes including "unsupported-trend-claim", "misclassified-tier",
  "stale-metrics", "recommendation-without-metric-anchor"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/channel-analyst.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: YouTubeAnalyst", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of analysis cards; right = selected-analysis detail with metrics summary,
  top-videos table, narrative summary, recommendation list, and eval-score chip).
  Browser title exactly: <title>Akka Sample: YouTubeAnalyst</title>. No subtitle on the
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(analysisId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    analyse-channel.json — 8 ChannelReport entries covering all three PerformanceTier values.
      Each entry has a summary paragraph referencing at least one numeric figure, a topVideos
      list with non-zero engagementRate values, and a non-empty recommendations list. Each
      Recommendation has a metricRef matching a field in ChannelMetrics. Plus 2 deliberately
      MALFORMED entries (one with a topVideos entry whose videoId is not in the fixture; one
      with a recommendations entry whose metricRef is "undefined") — the guardrail blocks
      both, exercising the retry path. The mock should select a malformed entry on the FIRST
      iteration of every 3rd analysis (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(analysisId) helper makes per-analysis selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChannelAnalystAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AnalysisTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (analyseStep
  60s, awaitMetricsStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ChannelAnalysis row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: AnalysisTasks.java with ANALYSE_CHANNEL = Task.name(...).description(...)
  .resultConformsTo(ChannelReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9308 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (ChannelAnalystAgent). The on-decision eval is rule-based (AnalysisEvaluator.java) and
  does NOT make an LLM call — keeping the pattern's "one agent" promise honest.
- The metrics are passed as a Task ATTACHMENT (JSON file), never inlined into the agent's
  instructions. Verify the generated analyseStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
