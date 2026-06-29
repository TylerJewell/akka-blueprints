# SPEC — financial-advisor-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Financial Advisor Pipeline.
**One-line pitch:** A user submits a financial question; one `FinancialAdvisorAgent` walks it through four task phases — **RESEARCH** market conditions, **STRATEGIZE** an approach, **PLAN** execution steps, **ASSESS** risk — with every outbound response guarded by a regulated disclaimer and a sector-compliance sanitizer.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a financial-advisory domain. One `FinancialAdvisorAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RESEARCH task's typed output becomes the STRATEGIZE task's instruction context; the STRATEGIZE task's typed output becomes the PLAN task's instruction context; the PLAN task's typed output informs the ASSESS task.

Two governance mechanisms are wired around the pipeline:

- A **`before-agent-response` guardrail** intercepts every outbound agent response before the workflow records it or the client reads it. It prepends a regulated disclaimer block — identifying the system as an educational tool, not a licensed advisor — and verifies that the disclaimer is present before the response proceeds. A response missing the disclaimer is held until the guardrail inserts it. The same hook also timestamps each disclaimer injection so the audit trail is complete.
- A **`sector` sanitizer** runs immediately after the guardrail on every outbound response. It scans the text for prohibited content patterns specific to the financial sector: guaranteed-return language (`"guaranteed", "risk-free", "certain return"`), specific securities recommendations phrased as directives (`"you must buy", "sell immediately"`), and unlicensed tax advice. Matched patterns are redacted with a `[REDACTED — compliance]` marker; the sanitizer records a `SanitizerFired` event on the entity for each match so the intervention is visible in the UI and in the audit log.

The blueprint shows that governance in a financial pipeline is not a single checkpoint — the disclaimer guardrail and the sector sanitizer are independent layers, each closing a distinct compliance gap.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **financial query** into the input (or picks one of three seeded queries — `retirement portfolio rebalancing for age 55`, `tax-loss harvesting strategy Q4`, `growth vs. value allocation 2026`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/advisories` and receives an `advisoryId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RESEARCHING` — the workflow has started `researchStep` and the agent has been handed the RESEARCH task.
4. Within ~15–25 s the card reaches `STRATEGIZING` — the typed `MarketSnapshot` is visible in the card detail (a table of market data points with sector, metric, and value). The agent's RESEARCH task returned; the workflow recorded `MarketResearched` and started the STRATEGIZE task.
5. Within ~15–25 s more the card reaches `PLANNING`. The `Strategy` is visible (approach summary, allocation targets, rationale items).
6. Within ~15–25 s more the card reaches `ASSESSING`. The `ExecutionPlan` is visible (ordered action items with timing and instrument targets).
7. Within ~15–25 s more the card reaches `EVALUATED`. The right pane now shows the full typed `FinancialReport` — title, summary, strategy, execution plan, risk profile — plus a compliance score chip (1–5) and a one-line rationale. The disclaimer-log strip shows every injected disclaimer with timestamp, and the sanitizer-event strip shows any redactions.
8. The user can submit another query; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AdvisoryEndpoint` | `HttpEndpoint` | `/api/advisories/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `AdvisoryEntity`, `AdvisoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `AdvisoryEntity` | `EventSourcedEntity` | Per-advisory lifecycle: created → researching → researched → strategizing → strategized → planning → planned → assessing → evaluated. Source of truth. | `AdvisoryEndpoint`, `AdvisorPipelineWorkflow` | `AdvisoryView` |
| `AdvisorPipelineWorkflow` | `Workflow` | One workflow per advisory. Steps: `researchStep` → `strategyStep` → `executionStep` → `riskStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `AdvisoryEndpoint` after `CREATED` | `FinancialAdvisorAgent`, `AdvisoryEntity` |
| `FinancialAdvisorAgent` | `AutonomousAgent` | The single agent. Declares four `Task<R>` constants in `AdvisorTasks.java`: `RESEARCH_MARKET` → `MarketSnapshot`, `DEFINE_STRATEGY` → `Strategy`, `PLAN_EXECUTION` → `ExecutionPlan`, `ASSESS_RISK` → `RiskProfile`. Each task is registered with the phase-appropriate function tools. | invoked by `AdvisorPipelineWorkflow` | returns typed results |
| `ResearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchMarketData(sector)` and `lookupBenchmark(index)`. Reads from `src/main/resources/sample-data/market/*.json` for deterministic offline output. | called from RESEARCH task | returns `List<MarketDataPoint>` |
| `StrategyTools` | function-tools class | Implements `buildAllocationTargets(snapshot)` and `scoreRationale(approach)`. Pure in-memory transformations. | called from STRATEGIZE task | returns `List<AllocationTarget>` / `List<RationaleItem>` |
| `ExecutionTools` | function-tools class | Implements `sequenceActions(strategy)` and `resolveInstruments(targets)`. | called from PLAN task | returns `List<ActionItem>` / `List<Instrument>` |
| `RiskTools` | function-tools class | Implements `measureVolatility(snapshot)` and `classifyRiskBand(profile)`. | called from ASSESS task | returns `VolatilityMetrics` / `RiskBand` |
| `DisclaimerGuardrail` | `before-agent-response` guardrail (registered on `FinancialAdvisorAgent`) | Prepends the regulated finance-advice disclaimer to every outbound agent response. Records `DisclaimerInjected` event on the entity. Rejects any response where disclaimer injection fails. | every outbound agent response | accept after injection / structured-reject |
| `SectorSanitizer` | sanitizer (flavor: sector, registered on `FinancialAdvisorAgent`) | Scans outbound text for prohibited financial-sector patterns. Redacts matches in-place. Records `SanitizerFired` event per match. | every outbound agent response (after guardrail) | redacted text / `SanitizerFired` event |
| `ComplianceScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `FinancialReport`, `RiskProfile`, `Strategy`. Output: `ComplianceResult{score, rationale}`. | called from `riskStep` eval sub-step | returns score |
| `AdvisoryView` | `View` | Read model: one row per advisory for the UI. | `AdvisoryEntity` events | `AdvisoryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MarketDataPoint(String sector, String metric, String value, String source, Instant capturedAt) {}

record MarketSnapshot(List<MarketDataPoint> dataPoints, String benchmarkIndex, double benchmarkReturn, Instant researchedAt) {}

record AllocationTarget(String assetClass, double targetPct, String rationale) {}

record RationaleItem(String itemId, String text, String supportingMetric) {}

record Strategy(
    String approach,
    List<AllocationTarget> allocations,
    List<RationaleItem> rationale,
    Instant definedAt
) {}

record Instrument(String ticker, String name, String assetClass) {}

record ActionItem(int sequence, String description, String timing, List<Instrument> instruments) {}

record ExecutionPlan(
    List<ActionItem> actions,
    String horizon,
    Instant plannedAt
) {}

record VolatilityMetrics(double portfolioStdDev, double beta, double sharpeRatio) {}

enum RiskBand { CONSERVATIVE, MODERATE, AGGRESSIVE }

record RiskProfile(
    RiskBand band,
    VolatilityMetrics volatility,
    List<String> mitigations,
    Instant assessedAt
) {}

record FinancialReport(
    String title,
    String summary,
    Strategy strategy,
    ExecutionPlan executionPlan,
    RiskProfile riskProfile,
    Instant reportedAt
) {}

record ComplianceResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record DisclaimerInjection(String advisoryId, String disclaimerText, Instant injectedAt) {}

record SanitizerEvent(String advisoryId, String matchedPattern, String redactedFragment, Instant firedAt) {}

record AdvisoryRecord(
    String advisoryId,
    Optional<String> query,
    Optional<MarketSnapshot> snapshot,
    Optional<Strategy> strategy,
    Optional<ExecutionPlan> executionPlan,
    Optional<RiskProfile> riskProfile,
    Optional<FinancialReport> report,
    Optional<ComplianceResult> compliance,
    AdvisoryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt,
    List<DisclaimerInjection> disclaimerLog,
    List<SanitizerEvent> sanitizerLog
) {}

enum AdvisoryStatus {
    CREATED, RESEARCHING, RESEARCHED, STRATEGIZING, STRATEGIZED,
    PLANNING, PLANNED, ASSESSING, EVALUATED, FAILED
}
```

Events on `AdvisoryEntity`: `AdvisoryCreated`, `ResearchStarted`, `MarketResearched`, `StrategyStarted`, `StrategyDefined`, `ExecutionStarted`, `ExecutionPlanned`, `RiskStarted`, `RiskAssessed`, `ReportAssembled`, `ComplianceScored`, `DisclaimerInjected`, `SanitizerFired`, `AdvisoryFailed`.

Every nullable lifecycle field on the `AdvisoryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/advisories` — body `{ query }` → `{ advisoryId }`.
- `GET /api/advisories` — list all advisories, newest-first.
- `GET /api/advisories/{id}` — one advisory.
- `GET /api/advisories/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Financial Advisor Pipeline</title>`.

The App UI tab is a two-column layout: a left rail with the live list of advisories (status pill + query + age) and a right pane with the selected advisory's detail — query, market snapshot table, strategy allocations, execution plan actions, risk profile, compliance score chip, disclaimer-log strip, and sanitizer-event strip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (disclaimer injection)**: `DisclaimerGuardrail` is registered on `FinancialAdvisorAgent` and runs before every outbound agent response reaches the workflow. It prepends a standard regulatory disclaimer block: *"This content is for educational purposes only and does not constitute licensed financial advice. Consult a qualified financial professional before making investment decisions."* The guardrail also records a `DisclaimerInjected{advisoryId, disclaimerText, injectedAt}` event on the entity so every injected notice is in the audit trail. If disclaimer injection fails for any reason, the response is held and the workflow step retries within its budget.
- **S1 — `sector` sanitizer (financial-sector compliance)**: `SectorSanitizer` is registered on `FinancialAdvisorAgent` and runs on every outbound response text after the disclaimer guardrail. It scans for three prohibited pattern classes: (1) guaranteed-return language (`guaranteed`, `risk-free`, `certain return`, `no-risk`); (2) directive securities recommendations (`you must buy`, `sell immediately`, `put everything into`); (3) unlicensed tax-advice directives (`you owe`, `you will owe`, `declare this as`). Each match is replaced with `[REDACTED — compliance]` and a `SanitizerFired{matchedPattern, redactedFragment}` event is recorded on the entity. The sanitizer is non-blocking — a response with redactions proceeds; only the match text is altered.

## 9. Agent prompts

- `FinancialAdvisorAgent` → `prompts/financial-advisor-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output. It is also instructed that all responses will have a disclaimer prepended by the runtime and that it must not invent disclaimer text itself.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded query `retirement portfolio rebalancing for age 55`; within 90 s the advisory reaches `EVALUATED` with a non-empty market snapshot, ≥ 2 allocation targets, ≥ 2 action items, a risk band, and a compliance score chip on the card.
2. **J2** — Every outbound agent response triggers `DisclaimerGuardrail`; the disclaimer-log strip on the selected advisory's right pane shows ≥ 4 entries (one per phase response) with timestamps.
3. **J3** — A mock-LLM trajectory produces a STRATEGIZE-phase response containing the phrase `guaranteed return`. `SectorSanitizer` redacts it; a `SanitizerFired` event lands on the entity; the sanitizer-event strip in the UI shows the matched pattern and the redacted fragment.
4. **J4** — A `FinancialReport` whose `RiskProfile` has no mitigations and whose `Strategy` has only one allocation target receives a compliance score ≤ 2 with a rationale naming the gap; the card border highlights red.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named financial-advisor-pipeline demonstrating the sequential-pipeline x
finance-analysis cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-finance-analysis-financial-advisor-pipeline.
Java package io.akka.samples.financialadvisor. Akka 3.6.0. HTTP port 9576.

Components to wire (exactly):

- 1 AutonomousAgent FinancialAdvisorAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/financial-advisor-agent.md>) and four .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the RESEARCH, STRATEGY, EXECUTION, and RISK tool sets are ALL registered
  on the agent. The before-agent-response guardrail (DisclaimerGuardrail) and the sector
  sanitizer (SectorSanitizer) are both registered on the agent via the agent's
  guardrail-configuration block.

- 1 Workflow AdvisorPipelineWorkflow per advisoryId with five steps (four task steps + error):
  * researchStep — emits ResearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(FinancialAdvisorAgent.class, "agent-" + advisoryId).runSingleTask(
      TaskDef.instructions("Query: " + query + "\nPhase: RESEARCH\nUse fetchMarketData and
      lookupBenchmark to gather market data points relevant to this query.")
        .metadata("advisoryId", advisoryId)
        .metadata("phase", "RESEARCH")
        .taskType(AdvisorTasks.RESEARCH_MARKET)
    ). Reads forTask(taskId).result(RESEARCH_MARKET) to get MarketSnapshot. Writes
    AdvisoryEntity.recordSnapshot(snapshot). WorkflowSettings.stepTimeout 70s.
  * strategyStep — emits StrategyStarted, then runSingleTask with TaskDef.instructions
    (formatStrategyContext(snapshot, query)) and metadata.phase = "STRATEGY", taskType
    DEFINE_STRATEGY. Writes AdvisoryEntity.recordStrategy(strategy). stepTimeout 70s.
  * executionStep — emits ExecutionStarted, then runSingleTask with TaskDef.instructions
    (formatExecutionContext(strategy, snapshot, query)) and metadata.phase = "EXECUTE", taskType
    PLAN_EXECUTION. Writes AdvisoryEntity.recordExecutionPlan(executionPlan). stepTimeout 70s.
  * riskStep — emits RiskStarted, then runSingleTask with TaskDef.instructions
    (formatRiskContext(executionPlan, strategy, snapshot, query)) and metadata.phase = "ASSESS",
    taskType ASSESS_RISK. Writes AdvisoryEntity.recordRiskProfile(riskProfile). Then assembles
    FinancialReport from the four typed results and writes AdvisoryEntity.recordReport(report).
    Then runs ComplianceScorer and writes AdvisoryEntity.recordCompliance(complianceResult).
    stepTimeout 70s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(AdvisorPipelineWorkflow::error). The error step writes
  AdvisoryFailed and ends.

- 1 EventSourcedEntity AdvisoryEntity (one per advisoryId). State AdvisoryRecord{advisoryId,
  query: Optional<String>, snapshot: Optional<MarketSnapshot>, strategy: Optional<Strategy>,
  executionPlan: Optional<ExecutionPlan>, riskProfile: Optional<RiskProfile>,
  report: Optional<FinancialReport>, compliance: Optional<ComplianceResult>,
  status: AdvisoryStatus, createdAt: Instant, finishedAt: Optional<Instant>,
  disclaimerLog: List<DisclaimerInjection>, sanitizerLog: List<SanitizerEvent>}.
  AdvisoryStatus enum: CREATED, RESEARCHING, RESEARCHED, STRATEGIZING, STRATEGIZED, PLANNING,
  PLANNED, ASSESSING, EVALUATED, FAILED. Events: AdvisoryCreated{query}, ResearchStarted,
  MarketResearched{snapshot}, StrategyStarted, StrategyDefined{strategy}, ExecutionStarted,
  ExecutionPlanned{executionPlan}, RiskStarted, RiskAssessed{riskProfile},
  ReportAssembled{report}, ComplianceScored{compliance}, DisclaimerInjected{advisoryId,
  disclaimerText, injectedAt}, SanitizerFired{matchedPattern, redactedFragment, firedAt},
  AdvisoryFailed{reason}.
  Commands: create, startResearch, recordSnapshot, startStrategy, recordStrategy,
  startExecution, recordExecutionPlan, startRisk, recordRiskProfile, recordReport,
  recordCompliance, recordDisclaimerInjection, recordSanitizerFired, fail, getAdvisory.
  emptyState() returns AdvisoryRecord.initial("") with all Optional fields as Optional.empty(),
  disclaimerLog = List.of(), sanitizerLog = List.of(), and no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View AdvisoryView with row type AdvisoryRow that mirrors AdvisoryRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes AdvisoryEntity events. ONE
  query getAllAdvisories: SELECT * AS advisories FROM advisory_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * AdvisoryEndpoint at /api with POST /advisories (body {query}; mints advisoryId; calls
    AdvisoryEntity.create(query); then starts AdvisorPipelineWorkflow with id
    "advisory-" + advisoryId; returns {advisoryId}), GET /advisories (list from
    getAllAdvisories, sorted newest-first), GET /advisories/{id} (one row), GET
    /advisories/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- AdvisorTasks.java declaring four Task<R> constants:
    RESEARCH_MARKET = Task.name("Research market").description("Gather market data points
      relevant to the query by calling fetchMarketData and lookupBenchmark")
      .resultConformsTo(MarketSnapshot.class);
    DEFINE_STRATEGY = Task.name("Define strategy").description("Build allocation targets and
      rationale items from the market snapshot").resultConformsTo(Strategy.class);
    PLAN_EXECUTION = Task.name("Plan execution").description("Sequence action items and resolve
      instruments from the strategy").resultConformsTo(ExecutionPlan.class);
    ASSESS_RISK = Task.name("Assess risk").description("Measure volatility and classify the
      risk band for the execution plan").resultConformsTo(RiskProfile.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- AdvisorPhase.java — enum {RESEARCH, STRATEGY, EXECUTE, ASSESS}. Each function-tool method is
  tagged with the constant phase via a parallel registry built at startup; the guardrail and
  sanitizer read it from that registry.

- ResearchTools.java — @FunctionTool fetchMarketData(String sector) -> List<MarketDataPoint>
  reading from src/main/resources/sample-data/market/<sector>.json; @FunctionTool
  lookupBenchmark(String index) -> double reading the benchmarkReturn for the named index.

- StrategyTools.java — @FunctionTool buildAllocationTargets(MarketSnapshot) ->
  List<AllocationTarget> (one target per asset class present in the snapshot, weight derived
  from sector momentum scores); @FunctionTool scoreRationale(String approach) ->
  List<RationaleItem> (one item per risk dimension, text drawn from snapshot metrics).

- ExecutionTools.java — @FunctionTool sequenceActions(Strategy) -> List<ActionItem> (one action
  per allocation target, ordered by urgency weight); @FunctionTool resolveInstruments(
  List<AllocationTarget>) -> List<Instrument> (maps each asset class to a representative
  ticker from src/main/resources/sample-data/instruments.json).

- RiskTools.java — @FunctionTool measureVolatility(MarketSnapshot) -> VolatilityMetrics
  (deterministic from sector std-dev data in the snapshot JSON); @FunctionTool
  classifyRiskBand(RiskProfile) -> RiskBand (CONSERVATIVE / MODERATE / AGGRESSIVE based on
  portfolioStdDev thresholds: < 0.08 → CONSERVATIVE, < 0.15 → MODERATE, >= 0.15 → AGGRESSIVE).

- DisclaimerGuardrail.java — implements the before-agent-response hook. Prepends the standard
  regulated disclaimer text to every outbound response text. Calls
  AdvisoryEntity.recordDisclaimerInjection(advisoryId, disclaimerText, now()) so the injection
  is visible in the disclaimer-log strip. Returns the modified response text. On failure
  (disclaimer text empty or injection call fails) returns Guardrail.reject("disclaimer-failure:
  could not inject disclaimer for advisoryId <id>").

- SectorSanitizer.java — implements the sector sanitizer hook. Runs after DisclaimerGuardrail
  on every outbound response text. Three prohibited pattern matchers (case-insensitive):
  (1) guaranteed-return: /guaranteed|risk.?free|certain return|no.?risk/,
  (2) directive-securities: /you must buy|sell immediately|put everything into/,
  (3) unlicensed-tax: /you owe|you will owe|declare this as/. Each match is replaced with
  "[REDACTED — compliance]". For each match, calls
  AdvisoryEntity.recordSanitizerFired(matchedPattern, redactedFragment, now()). Returns the
  redacted text. Non-blocking — a response with redactions proceeds.

- ComplianceScorer.java — pure deterministic logic (no LLM). Inputs: FinancialReport,
  RiskProfile, Strategy. Outputs: ComplianceResult with score and rationale. Four checks, one
  point per check satisfied, starting from a base of 1: risk-band present (RiskProfile.band
  is non-null), mitigations listed (RiskProfile.mitigations.size() >= 2), allocation breadth
  (Strategy.allocations.size() >= 2), source traceability (every RationaleItem.supportingMetric
  matches a metric from the MarketSnapshot). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9576 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/queries.jsonl with 5 seeded query lines covering the three
  surfaces named in J1–J4 plus two extras.

- src/main/resources/sample-data/market/*.json — three files keyed by sector (equities,
  fixed-income, alternatives), each carrying 6-10 MarketDataPoint entries with deterministic
  content so ResearchTools.fetchMarketData returns the same list across restarts.

- src/main/resources/sample-data/instruments.json — mapping of asset classes to representative
  Instrument entries (ticker, name, assetClass) for ExecutionTools.resolveInstruments.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root prefilled for the finance-analysis domain.

- prompts/financial-advisor-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Financial Advisor Pipeline",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of advisory cards; right = selected-advisory detail with query header, market
  snapshot table, strategy allocations, execution plan actions, risk profile band, compliance
  score chip, disclaimer-log strip, sanitizer-event strip). Browser title exactly:
  <title>Akka Sample: Financial Advisor Pipeline</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        typed-correct outputs per Task.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json and deserialises into the typed return.
- Per-task mock-response shapes:
    research-market.json — 5 MarketSnapshot entries, each with 6-8 MarketDataPoint items.
    define-strategy.json — 5 Strategy entries paired one-to-one, each with 2-4
      AllocationTarget items and 3-5 RationaleItem items. One entry contains the prohibited
      phrase "guaranteed return" in a RationaleItem.text — the SectorSanitizer redacts it;
      J3 verifies this.
    plan-execution.json — 5 ExecutionPlan entries, each with 2-4 ActionItem items.
    assess-risk.json — 5 RiskProfile entries, one per paired MarketSnapshot, each with a
      computed RiskBand and 2-4 mitigation strings. One entry has mitigations = [] and
      allocations.size() = 1 — ComplianceScorer scores it <= 2; J4 verifies this.
- MockModelProvider.seedFor(advisoryId) makes per-advisory selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: FinancialAdvisorAgent extends akka.javasdk.agent.autonomous.AutonomousAgent.
  The companion AdvisorTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (researchStep 70s, strategyStep
  70s, executionStep 70s, riskStep 70s, error 5s).
- Lesson 6: every nullable lifecycle field on AdvisoryRecord is Optional<T>.
- Lesson 7: AdvisorTasks.java with RESEARCH_MARKET, DEFINE_STRATEGY, PLAN_EXECUTION,
  ASSESS_RISK constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9576 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and themeVariables.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute. Exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (FinancialAdvisorAgent). ComplianceScorer
  is rule-based and does NOT make an LLM call.
- Sequential-pipeline invariant: all four tool sets registered on the agent; the guardrail and
  sanitizer are the output-side governance layer, not a tool-access gate.
- Task dependency is carried by typed task results: researchStep writes MarketSnapshot,
  strategyStep reads it, executionStep reads Strategy, riskStep reads all three. The agent is
  stateless across phases.
- Overview tab Try-it card shows just "/akka:build". No env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
