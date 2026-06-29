# SPEC — financial-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Financial Assistant (Multi-Agent).
**One-line pitch:** Submit a financial query; a coordinator delegates market research, portfolio planning, and report drafting to three specialist agents in parallel, then synthesises a single consolidated financial report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to three AutonomousAgents in parallel, gathers their results, and asks a fourth AutonomousAgent to synthesise a consolidated report. The blueprint also demonstrates a **finance-sector sanitizer** that scrubs investment-advice language before the report is committed, and a **before-agent-response guardrail** that vets the coordinator's synthesis for regulatory compliance. An eval-event governance control samples synthesis decisions for quality scoring.

## 3. User-facing flows

The user opens the App UI tab and submits a financial query (e.g. "Analyse NVDA risk/return for a growth portfolio").

1. The system creates a `FinancialReport` record in `DRAFTING` and starts a `FinancialWorkflow`.
2. The Coordinator decomposes the query into three parallel work items: a market-data query for the Researcher, a risk/return question for the Planner, and a narrative framing for the Drafter.
3. The workflow forks: all three agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the three payloads into a `ConsolidatedReport { executive_summary, market_section, planning_section, narrative, sanitizerVerdict, guardrailVerdict }`.
5. A finance-sector sanitizer scrubs the consolidated report for regulated phrases (e.g. "guaranteed return", "risk-free"); flagged phrases are replaced with compliant alternatives. An output guardrail then vets the result; if it fails, the report moves to `BLOCKED`.
6. On success the report moves to `PUBLISHED`.
7. If any specialist times out after 60 seconds, the workflow short-circuits: the Coordinator synthesises from whichever sides returned, and the report enters `DEGRADED`.

A `QuerySimulator` (TimedAction) drips a sample financial query every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `FinancialCoordinator` | `AutonomousAgent` | Decomposes the query into work items; synthesises the merged report; runs the before-agent-response guardrail. | `FinancialWorkflow` | returns typed result to workflow |
| `MarketResearcher` | `AutonomousAgent` | Gathers market data, price context, and sector indicators for the query. | `FinancialWorkflow` | — |
| `PortfolioPlanner` | `AutonomousAgent` | Evaluates risk/return characteristics; proposes allocation guidance. | `FinancialWorkflow` | — |
| `ReportDrafter` | `AutonomousAgent` | Writes the investor-ready narrative section from the coordinator's synthesis framing. | `FinancialWorkflow` | — |
| `FinancialWorkflow` | `Workflow` | Coordinates the parallel fan-out, synthesis, sanitizer, and guardrail. | `FinancialEndpoint`, `QueryRequestConsumer` | `FinancialReportEntity` |
| `FinancialReportEntity` | `EventSourcedEntity` | Holds the report's lifecycle (drafting → in-review → published / degraded / blocked). | `FinancialWorkflow` | `FinancialView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted query for replay/audit. | `FinancialEndpoint`, `QuerySimulator` | `QueryRequestConsumer` |
| `FinancialView` | `View` | List-of-reports read model. | `FinancialReportEntity` events | `FinancialEndpoint` |
| `QueryRequestConsumer` | `Consumer` | Listens to `QueryQueue` events and starts a workflow per submission. | `QueryQueue` events | `FinancialWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample financial query every 60 s. | scheduler | `QueryQueue` |
| `EvalSampler` | `TimedAction` | Samples one published report every 5 minutes for eval scoring; emits a `ReportEvalScored` event. | scheduler | `FinancialReportEntity` |
| `FinancialEndpoint` | `HttpEndpoint` | `/api/financial/*` — submit, get, list, SSE. | — | `FinancialView`, `QueryQueue`, `FinancialReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QueryRequest(String query, String requestedBy) {}

record MarketDataBundle(List<MarketDataPoint> dataPoints, Instant gatheredAt) {}
record MarketDataPoint(String ticker, String metric, String value, String source) {}

record PlanningAssessment(String riskProfile, List<String> allocationGuidance,
                          String rationale, Instant assessedAt) {}

record ReportNarrative(String narrativeText, List<String> keyMessages, Instant draftedAt) {}

record WorkAssignment(String marketQuery, String planningQuestion, String narrativeFraming) {}

record ConsolidatedReport(
    String executiveSummary,
    MarketDataBundle marketSection,
    PlanningAssessment planningSection,
    ReportNarrative narrative,
    String sanitizerVerdict,
    String guardrailVerdict,
    Instant synthesisedAt
) {}

record FinancialReport(
    String reportId,
    String query,
    ReportStatus status,
    Optional<MarketDataBundle> marketData,
    Optional<PlanningAssessment> planningAssessment,
    Optional<ReportNarrative> narrative,
    Optional<ConsolidatedReport> consolidated,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { DRAFTING, IN_REVIEW, PUBLISHED, DEGRADED, BLOCKED }
```

### Events (on `FinancialReportEntity`)

`ReportCreated`, `MarketDataAttached`, `PlanningAttached`, `NarrativeAttached`, `ReportPublished`, `ReportDegraded`, `ReportBlocked`, `ReportEvalScored`.

### Events (on `QueryQueue`)

`QuerySubmitted { reportId, query, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/financial` — body `{ query }` → `{ reportId }`. Starts a workflow.
- `GET /api/financial` — list all reports. Optional `?status=DRAFTING|IN_REVIEW|PUBLISHED|DEGRADED|BLOCKED`.
- `GET /api/financial/{id}` — one report.
- `GET /api/financial/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Financial Assistant (Multi-Agent)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a financial query, live list of reports with status pills, expand-row to see market data, planning assessment, narrative, consolidated summary, and eval score.

Browser title: `<title>Akka Sample: Financial Assistant (Multi-Agent)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — finance-sector sanitizer** (`sector` flavor on `FinancialCoordinator`): scrubs the consolidated report for regulated phrases before it is committed. Non-blocking by default — flagged phrases are replaced; only irremediable violations block the report.
- **G1 — before-agent-response guardrail** (`before-agent-response` on `FinancialCoordinator`): vets the full consolidated report for investment-advice compliance. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one published report every 5 minutes and emits a `ReportEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `FinancialCoordinator` → `prompts/coordinator.md`. Decomposes the query into work assignments; later synthesises results into the consolidated report.
- `MarketResearcher` → `prompts/market-researcher.md`. Gathers market data; returns `MarketDataBundle`.
- `PortfolioPlanner` → `prompts/portfolio-planner.md`. Evaluates risk/return; returns `PlanningAssessment`.
- `ReportDrafter` → `prompts/report-drafter.md`. Writes investor-ready narrative; returns `ReportNarrative`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a financial query; report progresses DRAFTING → IN_REVIEW → PUBLISHED within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `MarketResearcher` timeout to 1 s); report enters DEGRADED with whichever partial outputs came back.
3. **J3** — Inject a guardrail failure (Coordinator returns non-compliant investment advice language); report enters BLOCKED.
4. **J4** — Wait after a successful publish; the report's row in the UI shows an eval score.
5. **J5** — Leave the service running; `QuerySimulator` drips queries that flow through the full pipeline.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named financial-team demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-financial-team.
Java package io.akka.samples.financialassistantmultiagent. Akka 3.6.0. HTTP port 9919.

Components to wire (exactly):
- 4 AutonomousAgents:
  * FinancialCoordinator — definition() with
    capability(TaskAcceptance.of(ASSIGN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)).
    System prompt loaded from prompts/coordinator.md. Returns
    WorkAssignment{marketQuery, planningQuestion, narrativeFraming} for ASSIGN
    and ConsolidatedReport{executiveSummary, marketSection, planningSection,
    narrative, sanitizerVerdict, guardrailVerdict, synthesisedAt} for SYNTHESISE.
  * MarketResearcher — capability(TaskAcceptance.of(GATHER_MARKET).maxIterationsPerTask(3)).
    System prompt from prompts/market-researcher.md. Returns
    MarketDataBundle{dataPoints: List<MarketDataPoint{ticker, metric, value, source}>, gatheredAt}.
  * PortfolioPlanner — capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2)).
    System prompt from prompts/portfolio-planner.md. Returns
    PlanningAssessment{riskProfile, allocationGuidance: List<String>, rationale, assessedAt}.
  * ReportDrafter — capability(TaskAcceptance.of(DRAFT).maxIterationsPerTask(2)).
    System prompt from prompts/report-drafter.md. Returns
    ReportNarrative{narrativeText, keyMessages: List<String>, draftedAt}.

- 1 Workflow FinancialWorkflow with steps:
  assignStep -> [parallel] gatherMarketStep, planStep, draftStep -> joinStep ->
  synthesiseStep -> sanitizerStep -> guardrailStep -> emitStep.
  assignStep calls forAutonomousAgent(FinancialCoordinator.class, ASSIGN).
  gatherMarketStep, planStep, and draftStep run in parallel (CompletionStage
  allOf / zip chain); each governed by WorkflowSettings.builder()
  .stepTimeout(FinancialWorkflow::gatherMarketStep, ofSeconds(60))
  .stepTimeout(FinancialWorkflow::planStep, ofSeconds(60))
  .stepTimeout(FinancialWorkflow::draftStep, ofSeconds(60)).
  On any timeout, transition to a degradeStep that synthesises from whichever
  sides returned, then ends with ReportDegraded. synthesiseStep calls
  forAutonomousAgent(FinancialCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. sanitizerStep applies the
  finance-sector sanitizer (regex + replacement map for regulated phrases).
  guardrailStep runs the deterministic vetter + LLM judge on the consolidated
  report; on failure, end with ReportBlocked. WorkflowSettings is nested inside
  Workflow — no import.

- 1 EventSourcedEntity FinancialReportEntity holding state FinancialReport{reportId,
  query, ReportStatus, Optional<MarketDataBundle> marketData,
  Optional<PlanningAssessment> planningAssessment, Optional<ReportNarrative> narrative,
  Optional<ConsolidatedReport> consolidated, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: DRAFTING, IN_REVIEW, PUBLISHED,
  DEGRADED, BLOCKED. Events: ReportCreated, MarketDataAttached, PlanningAttached,
  NarrativeAttached, ReportPublished, ReportDegraded, ReportBlocked, ReportEvalScored.
  Commands: createReport, attachMarketData, attachPlanning, attachNarrative, publish,
  degrade, block, recordEval, getReport. emptyState() returns FinancialReport.initial
  with no commandContext() reference.

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(query, requestedBy)
  emitting QuerySubmitted{reportId, query, requestedBy, submittedAt}.

- 1 View FinancialView with row type FinancialReportRow (mirrors FinancialReport minus
  heavy nested payloads; every nullable field is Optional<T>). Table updater consumes
  FinancialReportEntity events. ONE query getAllReports SELECT * AS reports FROM
  financial_view. No WHERE status filter (Akka cannot auto-index enum columns) —
  caller filters client-side.

- 1 Consumer QueryRequestConsumer subscribed to QueryQueue events; on QuerySubmitted
  starts a FinancialWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/financial-queries.jsonl and calls
    QueryQueue.enqueueQuery.
  * EvalSampler — every 5 minutes, queries FinancialView.getAllReports, picks the
    oldest PUBLISHED report without an evalScore, runs a 1–5 rubric judge over the
    consolidated content, then calls FinancialReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * FinancialEndpoint at /api with POST /financial, GET /financial, GET /financial/{id},
    GET /financial/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- FinancialTasks.java declaring five Task<R> constants: ASSIGN (WorkAssignment),
  GATHER_MARKET (MarketDataBundle), PLAN (PlanningAssessment), DRAFT (ReportNarrative),
  SYNTHESISE (ConsolidatedReport).
- Domain records WorkAssignment, MarketDataPoint, MarketDataBundle, PlanningAssessment,
  ReportNarrative, ConsolidatedReport.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9919 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/financial-queries.jsonl with 8 canned query lines
  (e.g. "Analyse NVDA Q3 earnings impact on growth portfolios",
  "Assess duration risk in 10-year Treasuries for a conservative allocation").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (S1 sector sanitizer,
  G1 before-agent-response guardrail, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = finance-analysis,
  decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/market-researcher.md, prompts/portfolio-planner.md,
  prompts/report-drafter.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Financial Assistant (Multi-Agent)",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams
  + click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + live list with status pills). Browser title exactly:
  <title>Akka Sample: Financial Assistant (Multi-Agent)</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
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
  application.conf, no secrets.yaml, no .akka/ file with key material.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class name
  (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  market-researcher.json, portfolio-planner.json, report-drafter.json), picks
  one entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either WorkAssignment or ConsolidatedReport objects.
      4–6 WorkAssignment entries (marketQuery + planningQuestion + narrativeFraming
      triples) and 4–6 ConsolidatedReport entries (each with a 80–150 word
      executiveSummary, a 2–4 dataPoint market section, a 2–4 guidance planning
      section, a 60–100 word narrative, sanitizerVerdict = "clean",
      guardrailVerdict = "ok").
    market-researcher.json — 4–6 MarketDataBundle entries, each with 3–5 data points
      (ticker, metric, value, and a believable source like "Bloomberg 2025-Q2",
      "S&P Capital IQ", "unsourced — knowledge").
    portfolio-planner.json — 4–6 PlanningAssessment entries, each with a riskProfile
      label (e.g. "moderate-growth"), 3–5 allocationGuidance bullets, and a 1-sentence
      rationale.
    report-drafter.json — 4–6 ReportNarrative entries, each with 80–120 word
      narrativeText and 3–4 keyMessages.
- A MockModelProvider.seedFor(reportId) helper makes the selection deterministic
  per report id so the same report produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion FinancialTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9919 in application.conf.
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
- Parallel workflow steps use CompletionStage allOf/zip chain, NOT sequential calls.
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
