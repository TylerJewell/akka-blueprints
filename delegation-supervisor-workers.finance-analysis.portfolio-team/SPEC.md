# SPEC — portfolio-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Portfolio Assistant (Multi-Agent).
**One-line pitch:** Submit a holdings set; a coordinator delegates position analysis and market-context gathering to two specialist agents in parallel, then consolidates a single investment-grade portfolio report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to consolidate a unified report. The blueprint also demonstrates a **sector sanitizer** that normalizes and validates the submitted sector tag before analysis begins, and an **output guardrail** that vets the consolidated report for direct investment recommendations before it is returned.

## 3. User-facing flows

The user opens the App UI tab and submits a holdings set via the form.

1. The sector sanitizer normalizes the submitted sector tag. If the tag maps to a prohibited sector, the request is rejected before any agent work begins.
2. The system creates a `PortfolioReport` record in `PLANNING` and starts a `PortfolioWorkflow`.
3. The PortfolioCoordinator decomposes the holdings set into two parallel work items: a position-metrics query for the HoldingsAnalyst, and a macro/sector-background question for the MarketContextAgent.
4. The workflow forks: both agents run concurrently. Each returns a typed payload.
5. The Coordinator consolidates the two payloads into a `ConsolidatedReport { executive, holdingsAssessment, marketContext, guardrailVerdict }`.
6. An output guardrail vets the consolidated report for explicit buy/sell/hold directives; if it fails, the report moves to `BLOCKED`. Otherwise, the report moves to `CONSOLIDATED`.
7. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to consolidate from whichever side returned, and the report enters `DEGRADED`.

A `HoldingsSimulator` (TimedAction) drips a sample holdings set every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PortfolioCoordinator` | `AutonomousAgent` | Decomposes the holdings set into a work plan, consolidates the merged report, runs the output guardrail. | `PortfolioWorkflow` | returns typed result to workflow |
| `HoldingsAnalyst` | `AutonomousAgent` | Evaluates individual position metrics and sector exposure. Seeded "position tool" returns canned metrics. | `PortfolioWorkflow` | — |
| `MarketContextAgent` | `AutonomousAgent` | Gathers macro-economic and sector background relevant to the submitted holdings. | `PortfolioWorkflow` | — |
| `PortfolioWorkflow` | `Workflow` | Coordinates the sanitizer check, parallel fan-out, consolidation, and output guardrail. | `PortfolioEndpoint`, `HoldingsConsumer` | `PortfolioReportEntity` |
| `PortfolioReportEntity` | `EventSourcedEntity` | Holds the report lifecycle (planning → in-progress → consolidated / degraded / blocked). | `PortfolioWorkflow` | `PortfolioView` |
| `HoldingsQueue` | `EventSourcedEntity` | Logs each submitted holdings set for replay/audit. | `PortfolioEndpoint`, `HoldingsSimulator` | `HoldingsConsumer` |
| `PortfolioView` | `View` | List-of-reports read model. | `PortfolioReportEntity` events | `PortfolioEndpoint` |
| `HoldingsConsumer` | `Consumer` | Listens to `HoldingsQueue` events and starts a workflow per submission. | `HoldingsQueue` events | `PortfolioWorkflow` |
| `HoldingsSimulator` | `TimedAction` | Drips a sample holdings set every 60 s. | scheduler | `HoldingsQueue` |
| `EvalSampler` | `TimedAction` | Samples one consolidated report every 5 minutes for eval scoring; emits a `ReportEvalScored` event. | scheduler | `PortfolioReportEntity` |
| `PortfolioEndpoint` | `HttpEndpoint` | `/api/portfolio/*` — submit, get, list, SSE. | — | `PortfolioView`, `HoldingsQueue`, `PortfolioReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record HoldingsRequest(String portfolioId, String sector, List<Holding> holdings, String submittedBy) {}
record Holding(String ticker, String name, double weight, double marketValue) {}

record PositionAssessment(List<PositionNote> notes, String sectorExposure, Instant assessedAt) {}
record PositionNote(String ticker, String observation, String riskLevel) {}

record MarketContext(String macroSummary, List<String> sectorHeadwinds, List<String> sectorTailwinds, Instant gatheredAt) {}

record AnalysisPlan(String positionQuery, String marketQuestion) {}

record ConsolidatedReport(String executive, PositionAssessment holdingsAssessment,
                          MarketContext marketContext,
                          String guardrailVerdict, Instant consolidatedAt) {}

record PortfolioReport(
    String reportId,
    String portfolioId,
    String sector,
    ReportStatus status,
    Optional<PositionAssessment> holdingsAssessment,
    Optional<MarketContext> marketContext,
    Optional<ConsolidatedReport> consolidated,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { PLANNING, IN_PROGRESS, CONSOLIDATED, DEGRADED, BLOCKED }
```

### Events (on `PortfolioReportEntity`)

`ReportCreated`, `HoldingsAssessed`, `MarketContextAttached`, `ReportConsolidated`, `ReportDegraded`, `ReportBlocked`, `ReportEvalScored`.

### Events (on `HoldingsQueue`)

`HoldingsSubmitted { reportId, portfolioId, sector, holdings, submittedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/portfolio` — body `{ portfolioId, sector, holdings }` → `{ reportId }`. Sanitizes sector, then starts a workflow.
- `GET /api/portfolio` — list all reports. Optional `?status=PLANNING|IN_PROGRESS|CONSOLIDATED|DEGRADED|BLOCKED`.
- `GET /api/portfolio/{id}` — one report.
- `GET /api/portfolio/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Portfolio Assistant (Multi-Agent)"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a holdings set (portfolioId, sector, and one or more tickers), live list of reports with status pills, expand-row to see holdings assessment + market context + consolidated executive summary + eval score.

Browser title: `<title>Akka Sample: Portfolio Assistant (Multi-Agent)</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — sector sanitizer** (`sector` flavor): normalizes the submitted sector tag against a canonical list before any agent work begins. Unknown or prohibited sectors are rejected with a 400 response. Non-blocking to overall throughput, but blocking per-request.
- **G1 — output guardrail** (`before-agent-response` on `PortfolioCoordinator`): vets the consolidated report for explicit buy/sell/hold directives and ungrounded price targets. Blocking. Failure → `BLOCKED`.

## 9. Agent prompts

- `PortfolioCoordinator` → `prompts/coordinator.md`. Decomposes holdings into a work plan; later consolidates worker reports into the final portfolio summary.
- `HoldingsAnalyst` → `prompts/holdings-analyst.md`. Evaluates position metrics and sector exposure; returns `PositionAssessment`.
- `MarketContextAgent` → `prompts/market-context.md`. Gathers macro and sector background; returns `MarketContext`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a holdings set with a valid sector; report progresses PLANNING → IN_PROGRESS → CONSOLIDATED within 60 s; UI reflects each transition via SSE.
2. **J2** — Submit with a prohibited sector tag; the sanitizer rejects the request before any workflow starts; UI shows an error message, no report row created.
3. **J3** — Inject a worker timeout (set `HoldingsAnalyst` timeout to 1 s); report enters DEGRADED with whichever partial output came back.
4. **J4** — Inject a guardrail failure (Coordinator returns a report with an explicit buy recommendation); report enters BLOCKED.
5. **J5** — Wait after a successful consolidation; the report's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named portfolio-team demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-portfolio-team.
Java package io.akka.samples.portfolioassistantmultiagent. Akka 3.6.0. HTTP port 9528.

Components to wire (exactly):
- 3 AutonomousAgents:
  * PortfolioCoordinator — definition() with capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(CONSOLIDATE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/coordinator.md. Returns AnalysisPlan{positionQuery, marketQuestion} for PLAN
    and ConsolidatedReport{executive, holdingsAssessment, marketContext, guardrailVerdict,
    consolidatedAt} for CONSOLIDATE.
  * HoldingsAnalyst — capability(TaskAcceptance.of(ASSESS).maxIterationsPerTask(3)). System prompt
    from prompts/holdings-analyst.md. Returns PositionAssessment{notes: List<PositionNote{ticker,
    observation, riskLevel}>, sectorExposure, assessedAt}.
  * MarketContextAgent — capability(TaskAcceptance.of(GATHER_CONTEXT).maxIterationsPerTask(2)).
    System prompt from prompts/market-context.md. Returns MarketContext{macroSummary,
    sectorHeadwinds: List<String>, sectorTailwinds: List<String>, gatheredAt}.

- 1 Workflow PortfolioWorkflow with steps:
  sanitizeStep -> planStep -> [parallel] assessStep, contextStep -> joinStep ->
  consolidateStep -> guardrailStep -> emitStep.
  sanitizeStep validates and normalizes the sector tag; on prohibited sector, end with
  ReportBlocked immediately (no agent work). planStep calls
  forAutonomousAgent(PortfolioCoordinator.class, PLAN).
  assessStep and contextStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(PortfolioWorkflow::assessStep, ofSeconds(60)) and
  stepTimeout(PortfolioWorkflow::contextStep, ofSeconds(60)). On either timeout, transition
  to a degradeStep that calls consolidateStep with whichever side returned, then ends with
  ReportDegraded. consolidateStep calls forAutonomousAgent(PortfolioCoordinator.class,
  CONSOLIDATE) with the merged inputs; give consolidateStep a 90s stepTimeout.
  guardrailStep runs the deterministic vetter + LLM judge on the consolidated report; on
  failure, end with ReportBlocked. WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity PortfolioReportEntity holding state PortfolioReport{reportId, portfolioId,
  sector, ReportStatus, Optional<PositionAssessment> holdingsAssessment,
  Optional<MarketContext> marketContext, Optional<ConsolidatedReport> consolidated,
  Optional<String> failureReason, Optional<Integer> evalScore, Optional<String> evalRationale,
  Instant createdAt, Optional<Instant> finishedAt}. ReportStatus enum: PLANNING, IN_PROGRESS,
  CONSOLIDATED, DEGRADED, BLOCKED. Events: ReportCreated, HoldingsAssessed,
  MarketContextAttached, ReportConsolidated, ReportDegraded, ReportBlocked, ReportEvalScored.
  Commands: createReport, attachAssessment, attachMarketContext, consolidate, degrade, block,
  recordEval, getReport. emptyState() returns PortfolioReport.initial("", "", "") with no
  commandContext() reference.

- 1 EventSourcedEntity HoldingsQueue with command enqueueHoldings(portfolioId, sector, holdings,
  submittedBy) emitting HoldingsSubmitted{reportId, portfolioId, sector, holdings, submittedBy,
  submittedAt}.

- 1 View PortfolioView with row type PortfolioReportRow (mirrors PortfolioReport minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes PortfolioReportEntity
  events. ONE query getAllReports SELECT * AS reports FROM portfolio_view.
  No WHERE status filter — caller filters client-side.

- 1 Consumer HoldingsConsumer subscribed to HoldingsQueue events; on HoldingsSubmitted starts a
  PortfolioWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * HoldingsSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-holdings.jsonl and calls
    HoldingsQueue.enqueueHoldings.
  * EvalSampler — every 5 minutes, queries PortfolioView.getAllReports, picks the oldest
    CONSOLIDATED report without an evalScore, runs a 1–5 rubric judge over the consolidated
    executive summary, then calls PortfolioReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * PortfolioEndpoint at /api with POST /portfolio, GET /portfolio, GET /portfolio/{id},
    GET /portfolio/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- PortfolioTasks.java declaring four Task<R> constants: PLAN (AnalysisPlan), ASSESS
  (PositionAssessment), GATHER_CONTEXT (MarketContext), CONSOLIDATE (ConsolidatedReport).
- Domain records AnalysisPlan, Holding, HoldingsRequest, PositionNote, PositionAssessment,
  MarketContext, ConsolidatedReport.
- SectorRegistry.java — static list of permitted sector tags; used by sanitizeStep.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9528 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-holdings.jsonl with 8 canned holdings entries.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 sector sanitizer,
  G1 output guardrail before-agent-response) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector.primary = finance-analysis,
  decisions.authority_level = recommend-only, data.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/coordinator.md, prompts/holdings-analyst.md, prompts/market-context.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Portfolio Assistant (Multi-Agent)",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration section.
  NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Portfolio Assistant (Multi-Agent)</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (coordinator.json,
  holdings-analyst.json, market-context.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    coordinator.json — list of either AnalysisPlan or ConsolidatedReport objects.
      4–6 AnalysisPlan entries (positionQuery + marketQuestion pairs targeting
      realistic finance topics) and 4–6 ConsolidatedReport entries (each with a
      100–160 word executive summary grounded in the holdings, a
      PositionAssessment, a MarketContext, guardrailVerdict = "ok").
    holdings-analyst.json — 4–6 PositionAssessment entries, each with 3–6
      PositionNote entries (ticker, observation, riskLevel = LOW|MEDIUM|HIGH)
      and a sectorExposure summary string.
    market-context.json — 4–6 MarketContext entries, each with a two-sentence
      macroSummary, 2–4 sectorHeadwinds, and 2–4 sectorTailwinds.
- A MockModelProvider.seedFor(reportId) helper makes the selection deterministic
  per report id so the same report produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s consolidation); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion PortfolioTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9528 in application.conf.
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
