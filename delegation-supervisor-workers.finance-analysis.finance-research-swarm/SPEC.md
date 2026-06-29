# SPEC — finance-research-swarm

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Finance Assistant Swarm Agent.
**One-line pitch:** Type a company name or ticker; a lead coordinator delegates price, news, fundamentals, and ticker resolution to four specialist agents in parallel, then merges the results into one integrated stock report with governance guardrails.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to four AutonomousAgents in parallel, gathers their results, and asks a fifth AutonomousAgent to synthesise an integrated stock report. The blueprint also demonstrates:

- An **output guardrail** (`before-agent-response`) that vets the report for unsupported price claims and disallowed recommendation language before it is returned.
- A **sector sanitizer** (`sector`) that scrubs finance-sector compliance phrases from the user-facing summary.
- An **eval-event** sampler (`on-decision-eval`) that checks numeric accuracy of synthesised price and fundamental claims.

## 3. User-facing flows

The user opens the App UI tab and submits a query (company name or ticker) via the form.

1. The system creates a `StockReport` record in `RESOLVING` and starts a `ReportWorkflow`.
2. `ReportCoordinator` first tasks `TickerResolver` to normalise the input into a canonical ticker and exchange.
3. With the resolved ticker in hand, the workflow forks four workers in parallel: `PriceAnalyst`, `NewsAnalyst`, and `FundamentalsAnalyst` all run concurrently. `TickerResolver`'s result is passed to each.
4. The Coordinator merges all four worker payloads into an `IntegratedReport { summary, priceSummary, newsSummary, fundamentalsSummary, sanitizerVerdict, guardrailVerdict }`.
5. The sector sanitizer scrubs the merged summary; then an output guardrail vets numeric accuracy. If either check fails, the report moves to `BLOCKED`. Otherwise, the report moves to `PUBLISHED`.
6. If any worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to merge from whichever workers returned, and the report enters `PARTIAL`.

A `TickerSimulator` (TimedAction) drips a sample ticker query every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReportCoordinator` | `AutonomousAgent` | Decomposes the query into a resolution plan and work items; merges all worker outputs; runs the synthesis guardrail. | `ReportWorkflow` | returns typed result to workflow |
| `TickerResolver` | `AutonomousAgent` | Normalises a company name or partial ticker into a canonical `TickerResolution`. Seeded symbol map for demo. | `ReportWorkflow` | — |
| `PriceAnalyst` | `AutonomousAgent` | Retrieves and interprets recent price data for the resolved ticker. Seeded price tool. | `ReportWorkflow` | — |
| `NewsAnalyst` | `AutonomousAgent` | Retrieves and summarises recent news headlines for the resolved ticker. Seeded news tool. | `ReportWorkflow` | — |
| `FundamentalsAnalyst` | `AutonomousAgent` | Retrieves and interprets key fundamental metrics (P/E, EPS, market cap). Seeded fundamentals tool. | `ReportWorkflow` | — |
| `ReportWorkflow` | `Workflow` | Coordinates the resolution step, the parallel four-way fan-out, the merge, the sanitizer, and the guardrail. | `ReportEndpoint`, `ReportRequestConsumer` | `StockReportEntity` |
| `StockReportEntity` | `EventSourcedEntity` | Holds the report's lifecycle (resolving → in-progress → published / partial / blocked). | `ReportWorkflow` | `StockReportView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted query for replay/audit. | `ReportEndpoint`, `TickerSimulator` | `ReportRequestConsumer` |
| `StockReportView` | `View` | List-of-reports read model. | `StockReportEntity` events | `ReportEndpoint` |
| `ReportRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `ReportWorkflow` |
| `TickerSimulator` | `TimedAction` | Drips a sample ticker query every 60 s. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one published report every 5 minutes for numeric-accuracy eval; emits an `EvalScored` event. | scheduler | `StockReportEntity` |
| `ReportEndpoint` | `HttpEndpoint` | `/api/reports/*` — submit, get, list, SSE, metadata. | — | `StockReportView`, `RequestQueue`, `StockReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and root redirect. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record TickerResolution(String ticker, String exchange, String companyName) {}

record PricePoint(String date, double open, double close, double high, double low, long volume) {}
record PriceSummary(String ticker, List<PricePoint> recentPrices, double changePercent,
                   String trend, Instant retrievedAt) {}

record NewsItem(String headline, String source, String publishedAt, String snippet) {}
record NewsSummary(String ticker, List<NewsItem> headlines, String sentimentLabel,
                  Instant retrievedAt) {}

record FundamentalsSnapshot(String ticker, double peRatio, double eps,
                            long marketCapUsd, double revenueGrowthPercent,
                            Instant retrievedAt) {}

record IntegratedReport(
    String summary,
    PriceSummary priceSummary,
    NewsSummary newsSummary,
    FundamentalsSnapshot fundamentals,
    String sanitizerVerdict,
    String guardrailVerdict,
    Instant integratedAt
) {}

record StockReport(
    String reportId,
    String query,
    Optional<TickerResolution> ticker,
    ReportStatus status,
    Optional<PriceSummary> priceSummary,
    Optional<NewsSummary> newsSummary,
    Optional<FundamentalsSnapshot> fundamentals,
    Optional<IntegratedReport> integrated,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { RESOLVING, IN_PROGRESS, PUBLISHED, PARTIAL, BLOCKED }
```

### Events (on `StockReportEntity`)

`ReportCreated`, `TickerResolved`, `PriceAttached`, `NewsAttached`, `FundamentalsAttached`, `ReportPublished`, `ReportPartial`, `ReportBlocked`, `EvalScored`.

### Events (on `RequestQueue`)

`QuerySubmitted { reportId, query, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reports` — body `{ query }` → `{ reportId }`. Starts a workflow.
- `GET /api/reports` — list all reports. Optional `?status=RESOLVING|IN_PROGRESS|PUBLISHED|PARTIAL|BLOCKED`.
- `GET /api/reports/{id}` — one report.
- `GET /api/reports/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Finance Assistant Swarm Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a ticker query, live list of reports with status pills, expand-row to see price summary, news summary, fundamentals, the integrated report, and the eval score.

Browser title: `<title>Akka Sample: Finance Assistant Swarm Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `ReportCoordinator`): vets the integrated report for unsupported price claims, disallowed buy/sell recommendation language, and missing disclaimers. Blocking. Failure → `BLOCKED`.
- **S1 — sector sanitizer** (`sector`): scrubs the integrated summary for compliance phrases that are prohibited in investment-adjacent content (e.g., "guaranteed return", "certain profit"). Applied before the report is surfaced. Blocking.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one published report every 5 minutes, checks numeric accuracy of price and fundamental claims against the raw worker outputs, and emits an `EvalScored` event with a 1–5 score and rationale.

## 9. Agent prompts

- `ReportCoordinator` → `prompts/report-coordinator.md`. Decomposes the query; synthesises worker results into the integrated report.
- `TickerResolver` → `prompts/ticker-resolver.md`. Normalises company name or ticker to canonical symbol and exchange.
- `PriceAnalyst` → `prompts/price-analyst.md`. Interprets recent price data; returns `PriceSummary`.
- `NewsAnalyst` → `prompts/news-analyst.md`. Retrieves and summarises headlines; returns `NewsSummary`.
- `FundamentalsAnalyst` → `prompts/fundamentals-analyst.md`. Interprets key fundamentals; returns `FundamentalsSnapshot`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a ticker query; report progresses RESOLVING → IN_PROGRESS → PUBLISHED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `PriceAnalyst` timeout to 1 s); report enters PARTIAL with whichever outputs came back.
3. **J3** — Inject a guardrail failure (Coordinator returns a "guaranteed return" phrase); report enters BLOCKED.
4. **J4** — Wait after a successful publish; the report's row in the UI shows an eval score.
5. **J5** — Background simulator drips a ticker every 60 s; App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named finance-research-swarm demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-finance-research-swarm.
Java package io.akka.samples.financeassistantswarmagent. Akka 3.6.0. HTTP port 9351.

Components to wire (exactly):
- 5 AutonomousAgents:
  * ReportCoordinator — definition() with
      capability(TaskAcceptance.of(PLAN_QUERY).maxIterationsPerTask(2))
      AND capability(TaskAcceptance.of(INTEGRATE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/report-coordinator.md.
    Returns QueryPlan{tickerQuery, priceQuestion, newsQuestion, fundamentalsQuestion}
    for PLAN_QUERY and IntegratedReport{summary, priceSummary, newsSummary,
    fundamentals, sanitizerVerdict, guardrailVerdict, integratedAt} for INTEGRATE.
  * TickerResolver — capability(TaskAcceptance.of(RESOLVE).maxIterationsPerTask(2)).
    System prompt from prompts/ticker-resolver.md.
    Returns TickerResolution{ticker, exchange, companyName}.
  * PriceAnalyst — capability(TaskAcceptance.of(ANALYSE_PRICE).maxIterationsPerTask(3)).
    System prompt from prompts/price-analyst.md.
    Returns PriceSummary{ticker, recentPrices: List<PricePoint{date, open, close,
    high, low, volume}>, changePercent, trend, retrievedAt}.
  * NewsAnalyst — capability(TaskAcceptance.of(ANALYSE_NEWS).maxIterationsPerTask(3)).
    System prompt from prompts/news-analyst.md.
    Returns NewsSummary{ticker, headlines: List<NewsItem{headline, source,
    publishedAt, snippet}>, sentimentLabel, retrievedAt}.
  * FundamentalsAnalyst — capability(TaskAcceptance.of(ANALYSE_FUNDAMENTALS).maxIterationsPerTask(3)).
    System prompt from prompts/fundamentals-analyst.md.
    Returns FundamentalsSnapshot{ticker, peRatio, eps, marketCapUsd,
    revenueGrowthPercent, retrievedAt}.

- 1 Workflow ReportWorkflow with steps:
  planStep -> resolveStep -> [parallel] priceStep, newsStep, fundamentalsStep
           -> joinStep -> integrateStep -> sanitizerStep -> guardrailStep -> emitStep.
  planStep calls forAutonomousAgent(ReportCoordinator.class, PLAN_QUERY).
  resolveStep calls forAutonomousAgent(TickerResolver.class, RESOLVE).
  priceStep, newsStep, and fundamentalsStep run in parallel (CompletionStage zip);
  each governed by WorkflowSettings.builder().stepTimeout(ReportWorkflow::priceStep, ofSeconds(60)),
  stepTimeout(ReportWorkflow::newsStep, ofSeconds(60)),
  stepTimeout(ReportWorkflow::fundamentalsStep, ofSeconds(60)).
  On any timeout, transition to a partialStep that calls integrateStep with whichever
  workers returned, then ends with ReportPartial.
  integrateStep calls forAutonomousAgent(ReportCoordinator.class, INTEGRATE) with the
  merged inputs; give integrateStep a 90s stepTimeout.
  sanitizerStep runs the deterministic sector-phrase scrubber over the integrated summary;
  on failure, end with ReportBlocked.
  guardrailStep runs the deterministic numeric-accuracy vetter + LLM judge on the
  integrated content; on failure, end with ReportBlocked. WorkflowSettings is nested
  inside Workflow — no import.

- 1 EventSourcedEntity StockReportEntity holding state StockReport{reportId, query,
  Optional<TickerResolution> ticker, ReportStatus, Optional<PriceSummary> priceSummary,
  Optional<NewsSummary> newsSummary, Optional<FundamentalsSnapshot> fundamentals,
  Optional<IntegratedReport> integrated, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: RESOLVING, IN_PROGRESS, PUBLISHED,
  PARTIAL, BLOCKED. Events: ReportCreated, TickerResolved, PriceAttached, NewsAttached,
  FundamentalsAttached, ReportPublished, ReportPartial, ReportBlocked, EvalScored.
  Commands: createReport, resolveTicker, attachPrice, attachNews, attachFundamentals,
  publish, partial, block, recordEval, getReport. emptyState() returns
  StockReport.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueQuery(query, requestedBy)
  emitting QuerySubmitted{reportId, query, requestedBy, submittedAt}.

- 1 View StockReportView with row type StockReportRow (mirrors StockReport minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  StockReportEntity events. ONE query getAllReports SELECT * AS reports FROM report_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer ReportRequestConsumer subscribed to RequestQueue events; on QuerySubmitted
  starts a ReportWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * TickerSimulator — every 60s, reads next line from
    src/main/resources/sample-events/ticker-queries.jsonl and calls
    RequestQueue.enqueueQuery.
  * EvalSampler — every 5 minutes, queries StockReportView.getAllReports, picks the
    oldest PUBLISHED report without an evalScore, runs a 1–5 numeric-accuracy rubric
    judge over the integrated content against the raw worker outputs, then calls
    StockReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * ReportEndpoint at /api with POST /reports, GET /reports, GET /reports/{id},
    GET /reports/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- FinanceTasks.java declaring five Task<R> constants: PLAN_QUERY (QueryPlan), RESOLVE
  (TickerResolution), ANALYSE_PRICE (PriceSummary), ANALYSE_NEWS (NewsSummary),
  ANALYSE_FUNDAMENTALS (FundamentalsSnapshot), INTEGRATE (IntegratedReport).
- Domain records QueryRequest, QueryPlan, TickerResolution, PricePoint, PriceSummary,
  NewsItem, NewsSummary, FundamentalsSnapshot, IntegratedReport.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9351 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/ticker-queries.jsonl with 8 canned query lines
  covering well-known tickers (AAPL, MSFT, GOOGL, AMZN, NVDA, TSLA, META, JPM).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (G1 output guardrail
  before-agent-response, S1 sanitizer sector, E1 eval-event on-decision-eval) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = finance-analysis, decisions
  = stock-report recommendations, data.pii = false, capabilities.* per domain;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/report-coordinator.md, prompts/ticker-resolver.md, prompts/price-analyst.md,
  prompts/news-analyst.md, prompts/fundamentals-analyst.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Finance Assistant Swarm Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams
  + click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Finance Assistant Swarm Agent</title>. No subtitle on the
  Overview tab.

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
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json, picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    report-coordinator.json — list of either QueryPlan or IntegratedReport objects.
      4–6 QueryPlan entries (priceQuestion + newsQuestion + fundamentalsQuestion
      triples referencing the resolved ticker) and 4–6 IntegratedReport entries
      (each with an 80–150 word summary, full priceSummary, newsSummary, fundamentals
      stubs, sanitizerVerdict = "ok", guardrailVerdict = "ok").
    ticker-resolver.json — 8 TickerResolution entries covering AAPL/NASDAQ,
      MSFT/NASDAQ, GOOGL/NASDAQ, AMZN/NASDAQ, NVDA/NASDAQ, TSLA/NASDAQ,
      META/NASDAQ, JPM/NYSE.
    price-analyst.json — 4–6 PriceSummary entries, each with 5 PricePoint objects
      with plausible open/close/high/low/volume values and a trend label
      ("upward", "downward", "sideways").
    news-analyst.json — 4–6 NewsSummary entries, each with 3–5 NewsItem objects
      with plausible headlines, sources (e.g. "Reuters", "Bloomberg", "WSJ"),
      and sentimentLabel values ("positive", "negative", "neutral").
    fundamentals-analyst.json — 4–6 FundamentalsSnapshot entries with plausible
      peRatio (10–50), eps (0.5–15), marketCapUsd (1e10–3e12), revenueGrowthPercent
      (-5–40).
- A MockModelProvider.seedFor(reportId) helper makes the selection deterministic
  per report id so the same report produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s integration); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion FinanceTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9351 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
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
