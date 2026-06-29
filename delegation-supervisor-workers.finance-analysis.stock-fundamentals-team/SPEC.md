# SPEC — stock-fundamentals-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** AI Team for Fundamental Stock Analysis.
**One-line pitch:** Submit a ticker; a coordinator delegates financials, news, and ratio analysis to three specialist agents in parallel, then synthesises a single stock recommendation.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their results, and asks a fourth AutonomousAgent to synthesise a recommendation. The blueprint also demonstrates an **output guardrail** that enforces a no-direct-advice disclaimer before the recommendation is surfaced, a **sector sanitizer** that redacts non-public price-sensitive language, and an **eval-event** control that scores the coordinator's synthesis decision for factual accuracy.

## 3. User-facing flows

The user opens the App UI tab and submits a stock ticker via the form.

1. The system creates a `StockReport` record in `QUEUED` and starts an `AnalysisWorkflow`.
2. The Coordinator decomposes the ticker request into three parallel subtasks: a financials query for `FinancialsAgent`, a news query for `NewsAgent`, and a ratios query for `RatioAgent`.
3. The workflow forks: all three agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the three payloads into a `StockRecommendation { ticker, summary, financialHighlights, newsSentiment, ratioAnalysis, stance, disclaimerPresent }`.
5. An output guardrail vets the recommendation for direct investment advice; if it triggers, the report moves to `BLOCKED`. A sector sanitizer redacts non-public price-sensitive text. If both pass, the report moves to `COMPLETE`.
6. If any worker times out after 60 seconds, the workflow short-circuits to a degrade path: the Coordinator synthesises from whichever agents returned, and the report enters `DEGRADED`.

A `TickerSimulator` (TimedAction) drips a sample ticker every 90 seconds so the App UI is non-empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AnalysisCoordinator` | `AutonomousAgent` | Decomposes the ticker request into subtasks; synthesises the merged recommendation; enforces the output guardrail. | `AnalysisWorkflow` | returns typed result to workflow |
| `FinancialsAgent` | `AutonomousAgent` | Extracts key financial metrics (revenue, EPS, operating income) from statement data. | `AnalysisWorkflow` | — |
| `NewsAgent` | `AutonomousAgent` | Summarises recent news headlines and assigns a sentiment score for the ticker. | `AnalysisWorkflow` | — |
| `RatioAgent` | `AutonomousAgent` | Computes and interprets standard valuation and profitability ratios (P/E, P/B, ROE, debt/equity). | `AnalysisWorkflow` | — |
| `AnalysisWorkflow` | `Workflow` | Coordinates the parallel fan-out, synthesis, guardrail, and sanitizer steps. | `AnalysisEndpoint`, `TickerRequestConsumer` | `StockReportEntity` |
| `StockReportEntity` | `EventSourcedEntity` | Holds the report lifecycle (queued → analysing → complete / degraded / blocked). | `AnalysisWorkflow` | `StockReportView` |
| `TickerQueue` | `EventSourcedEntity` | Logs each submitted ticker request for replay/audit. | `AnalysisEndpoint`, `TickerSimulator` | `TickerRequestConsumer` |
| `StockReportView` | `View` | List-of-reports read model. | `StockReportEntity` events | `AnalysisEndpoint` |
| `TickerRequestConsumer` | `Consumer` | Listens to `TickerQueue` events and starts a workflow per submission. | `TickerQueue` events | `AnalysisWorkflow` |
| `TickerSimulator` | `TimedAction` | Drips a sample ticker every 90 s. | scheduler | `TickerQueue` |
| `EvalSampler` | `TimedAction` | Samples one complete report every 5 minutes for eval scoring; emits a `ReportEvalScored` event. | scheduler | `StockReportEntity` |
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/analysis/*` — submit, get, list, SSE. | — | `StockReportView`, `TickerQueue`, `StockReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record TickerRequest(String ticker, String requestedBy) {}

record FinancialMetrics(
    String ticker,
    Optional<Double> revenueGrowthPct,
    Optional<Double> epsLatest,
    Optional<Double> operatingMarginPct,
    Instant extractedAt
) {}

record NewsItem(String headline, String source, String sentiment) {}
record NewsSummary(
    String ticker,
    List<NewsItem> items,
    String overallSentiment,
    Instant summarisedAt
) {}

record RatioSet(
    String ticker,
    Optional<Double> peRatio,
    Optional<Double> pbRatio,
    Optional<Double> roe,
    Optional<Double> debtEquityRatio,
    Instant computedAt
) {}

record AnalysisSubtasks(
    String financialsQuery,
    String newsQuery,
    String ratiosQuery
) {}

record StockRecommendation(
    String ticker,
    String summary,
    FinancialMetrics financialHighlights,
    NewsSummary newsSentiment,
    RatioSet ratioAnalysis,
    RecommendationStance stance,
    boolean disclaimerPresent,
    String sanitizerVerdict,
    Instant synthesisedAt
) {}

record StockReport(
    String reportId,
    String ticker,
    ReportStatus status,
    Optional<FinancialMetrics> financials,
    Optional<NewsSummary> news,
    Optional<RatioSet> ratios,
    Optional<StockRecommendation> recommendation,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { QUEUED, ANALYSING, COMPLETE, DEGRADED, BLOCKED }
enum RecommendationStance { POSITIVE, NEUTRAL, CAUTIOUS, INSUFFICIENT_DATA }
```

### Events (on `StockReportEntity`)

`ReportCreated`, `FinancialsAttached`, `NewsAttached`, `RatiosAttached`, `ReportCompleted`, `ReportDegraded`, `ReportBlocked`, `ReportEvalScored`.

### Events (on `TickerQueue`)

`TickerSubmitted { reportId, ticker, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/analysis` — body `{ ticker }` → `{ reportId }`. Starts a workflow.
- `GET /api/analysis` — list all reports. Optional `?status=QUEUED|ANALYSING|COMPLETE|DEGRADED|BLOCKED`.
- `GET /api/analysis/{id}` — one report.
- `GET /api/analysis/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "AI Team for Fundamental Stock Analysis"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a ticker, live list of reports with status pills, expand-row to see financials + news + ratios + synthesised recommendation + eval score.

Browser title: `<title>Akka Sample: AI Team for Fundamental Stock Analysis</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `AnalysisCoordinator`): vets the synthesised recommendation for direct investment advice and fabricated financial figures. Blocking. Failure → `BLOCKED`.
- **S1 — sector sanitizer** (`sector` flavor on `AnalysisCoordinator`): redacts any non-public price-sensitive language (insider-adjacent phrases, specific target prices framed as fact) before the recommendation is surfaced. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one complete report every 5 minutes and emits a `ReportEvalScored` event with a 1–5 factuality score and a short rationale.

## 9. Agent prompts

- `AnalysisCoordinator` → `prompts/coordinator.md`. Decomposes the ticker request; later synthesises the three worker results into a final recommendation.
- `FinancialsAgent` → `prompts/financials-agent.md`. Extracts revenue, EPS, and margin metrics; returns `FinancialMetrics`.
- `NewsAgent` → `prompts/news-agent.md`. Summarises headlines; assigns an overall sentiment; returns `NewsSummary`.
- `RatioAgent` → `prompts/ratio-agent.md`. Computes and interprets standard ratios; returns `RatioSet`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a ticker; report progresses QUEUED → ANALYSING → COMPLETE within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `FinancialsAgent` timeout to 1 s); report enters DEGRADED with partial output from the two remaining agents.
3. **J3** — Inject a guardrail failure (Coordinator returns a direct buy/sell instruction); report enters BLOCKED.
4. **J4** — Wait after a successful completion; the report's row in the UI shows an eval score.
5. **J5** — Sector sanitizer strips a non-public price-sensitive phrase; report still enters COMPLETE after sanitization passes.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named stock-fundamentals-team demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-stock-fundamentals-team.
Java package io.akka.samples.aiteamtoautomatefundamentalstockanalysis.
Akka 3.6.0. HTTP port 9437.

Components to wire (exactly):
- 4 AutonomousAgents:
  * AnalysisCoordinator — definition() with
    capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/coordinator.md.
    Returns AnalysisSubtasks{financialsQuery, newsQuery, ratiosQuery} for DECOMPOSE
    and StockRecommendation{ticker, summary, financialHighlights, newsSentiment,
    ratioAnalysis, stance, disclaimerPresent, sanitizerVerdict, synthesisedAt} for SYNTHESISE.
  * FinancialsAgent — capability(TaskAcceptance.of(EXTRACT_FINANCIALS).maxIterationsPerTask(3)).
    System prompt from prompts/financials-agent.md.
    Returns FinancialMetrics{ticker, revenueGrowthPct, epsLatest, operatingMarginPct, extractedAt}.
  * NewsAgent — capability(TaskAcceptance.of(GATHER_NEWS).maxIterationsPerTask(3)).
    System prompt from prompts/news-agent.md.
    Returns NewsSummary{ticker, items: List<NewsItem{headline, source, sentiment}>,
    overallSentiment, summarisedAt}.
  * RatioAgent — capability(TaskAcceptance.of(COMPUTE_RATIOS).maxIterationsPerTask(2)).
    System prompt from prompts/ratio-agent.md.
    Returns RatioSet{ticker, peRatio, pbRatio, roe, debtEquityRatio, computedAt}.

- 1 Workflow AnalysisWorkflow with steps:
  decomposeStep -> [parallel] financialsStep, newsStep, ratiosStep -> joinStep ->
  synthesiseStep -> guardrailStep -> sanitizerStep -> emitStep.
  decomposeStep calls forAutonomousAgent(AnalysisCoordinator.class, DECOMPOSE).
  financialsStep, newsStep, and ratiosStep run in parallel (CompletionStage allOf);
  each governed by WorkflowSettings.builder()
    .stepTimeout(AnalysisWorkflow::financialsStep, ofSeconds(60))
    .stepTimeout(AnalysisWorkflow::newsStep, ofSeconds(60))
    .stepTimeout(AnalysisWorkflow::ratiosStep, ofSeconds(60)).
  On any timeout, transition to a degradeStep that synthesises from whichever agents
  returned, then ends with ReportDegraded.
  synthesiseStep calls forAutonomousAgent(AnalysisCoordinator.class, SYNTHESISE) with
  the merged inputs; give synthesiseStep a 90s stepTimeout.
  guardrailStep runs a deterministic no-advice vetter + LLM judge; on failure end
  with ReportBlocked.
  sanitizerStep applies the sector sanitizer for price-sensitive language; on failure
  end with ReportBlocked.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity StockReportEntity holding state StockReport{reportId, ticker,
  ReportStatus, Optional<FinancialMetrics> financials, Optional<NewsSummary> news,
  Optional<RatioSet> ratios, Optional<StockRecommendation> recommendation,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  ReportStatus enum: QUEUED, ANALYSING, COMPLETE, DEGRADED, BLOCKED.
  RecommendationStance enum: POSITIVE, NEUTRAL, CAUTIOUS, INSUFFICIENT_DATA.
  Events: ReportCreated, FinancialsAttached, NewsAttached, RatiosAttached,
  ReportCompleted, ReportDegraded, ReportBlocked, ReportEvalScored.
  Commands: createReport, attachFinancials, attachNews, attachRatios, complete,
  degrade, block, recordEval, getReport.
  emptyState() returns StockReport.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity TickerQueue with command enqueueTicker(ticker, requestedBy)
  emitting TickerSubmitted{reportId, ticker, requestedBy, submittedAt}.

- 1 View StockReportView with row type StockReportRow (mirrors StockReport minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  StockReportEntity events. ONE query getAllReports SELECT * AS reports FROM report_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer TickerRequestConsumer subscribed to TickerQueue events; on TickerSubmitted
  starts an AnalysisWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * TickerSimulator — every 90s, reads next line from
    src/main/resources/sample-events/tickers.jsonl and calls TickerQueue.enqueueTicker.
  * EvalSampler — every 5 minutes, queries StockReportView.getAllReports, picks the oldest
    COMPLETE report without an evalScore, runs a 1–5 factuality rubric judge over the
    recommendation content, then calls StockReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /analysis, GET /analysis, GET /analysis/{id},
    GET /analysis/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- AnalysisTasks.java declaring five Task<R> constants: DECOMPOSE (AnalysisSubtasks),
  EXTRACT_FINANCIALS (FinancialMetrics), GATHER_NEWS (NewsSummary),
  COMPUTE_RATIOS (RatioSet), SYNTHESISE (StockRecommendation).
- Domain records AnalysisSubtasks, FinancialMetrics, NewsItem, NewsSummary,
  RatioSet, StockRecommendation.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9437 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/tickers.jsonl with 8 canned ticker lines
  (e.g., AAPL, MSFT, GOOGL, TSLA, NVDA, JPM, AMZN, META).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, S1 sanitizer sector, E1 eval-event on-decision-eval)
  and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = ["financial analysis",
  "investment research automation"], decisions.authority_level = recommend-only,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/financials-agent.md, prompts/news-agent.md,
  prompts/ratio-agent.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: AI Team for Fundamental Stock
  Analysis", one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: AI Team for Fundamental Stock Analysis</title>. No subtitle on
  the Overview tab.

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
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the Task<R> id.
  Each branch reads a JSON file from src/main/resources/mock-responses/ and
  picks one entry pseudo-randomly per call.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either AnalysisSubtasks or StockRecommendation objects.
      4–6 AnalysisSubtasks entries and 4–6 StockRecommendation entries (each with an
      80–150 word summary, fully populated financialHighlights, newsSentiment,
      ratioAnalysis, stance = one of the enum values, disclaimerPresent = true,
      sanitizerVerdict = "ok").
    financials-agent.json — 4–6 FinancialMetrics entries with realistic-looking
      growth percentages, EPS values, and operating margins (not real data).
    news-agent.json — 4–6 NewsSummary entries with 3–6 NewsItem each; sentiment
      values "positive", "neutral", or "negative"; overallSentiment one of same.
    ratio-agent.json — 4–6 RatioSet entries with believable P/E, P/B, ROE, and
      debt/equity values.
- A MockModelProvider.seedFor(reportId) helper makes the selection deterministic
  per report id.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s
  workers, 90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion AnalysisTasks.java declaring Task<R>
  constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9437 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme
  variables (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList
  index; delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage allOf, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
