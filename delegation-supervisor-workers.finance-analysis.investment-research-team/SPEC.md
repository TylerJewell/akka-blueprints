# SPEC — investment-research-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Investment Research Multi-Agent.
**One-line pitch:** Submit a ticker and research request; a coordinator delegates fundamental analysis to an Equity Analyst and market-context gathering to a Market Scout in parallel, then synthesises one investment research report with a sector-scoped content check.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise an investment report. The blueprint also demonstrates a **sector sanitizer** guardrail that scrubs regulated investment content before it is returned, and **eval-event** governance that samples the coordinator's synthesis decision for numeric-claim quality scoring.

## 3. User-facing flows

The user opens the App UI tab and submits a ticker symbol and a research question via the form.

1. The system creates a `ResearchReport` record in `PLANNING` and starts a `ReportWorkflow`.
2. The Coordinator decomposes the request into two parallel sub-tasks: a fundamental-data query for the EquityAnalyst and a market-context query for the MarketScout.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `SynthesisedReport { summary, fundamentals, sentiment, sanitizerVerdict }`.
5. A sector sanitizer vets the synthesised report; if it fails, the report moves to `BLOCKED`. Otherwise, the report moves to `PUBLISHED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to synthesise from whichever side returned, and the report enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample ticker research request every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ReportCoordinator` | `AutonomousAgent` | Decomposes the research request, synthesises the merged report, applies the sector sanitizer check. | `ReportWorkflow` | returns typed result to workflow |
| `EquityAnalyst` | `AutonomousAgent` | Gathers fundamental financial data (earnings, ratios, guidance). Seeded "market data tool" returns canned results. | `ReportWorkflow` | — |
| `MarketScout` | `AutonomousAgent` | Gathers market-sentiment and price-action context. | `ReportWorkflow` | — |
| `ReportWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, and the sanitizer step. | `ReportEndpoint`, `RequestConsumer` | `ResearchReportEntity` |
| `ResearchReportEntity` | `EventSourcedEntity` | Holds the report's lifecycle (planning → in-progress → published / degraded / blocked). | `ReportWorkflow` | `ReportView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted research request for replay/audit. | `ReportEndpoint`, `RequestSimulator` | `RequestConsumer` |
| `ReportView` | `View` | List-of-reports read model. | `ResearchReportEntity` events | `ReportEndpoint` |
| `RequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `ReportWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample research request every 60 s. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one published report every 5 minutes for numeric-claim quality scoring; emits a `ReportEvalScored` event. | scheduler | `ResearchReportEntity` |
| `ReportEndpoint` | `HttpEndpoint` | `/api/reports/*` — submit, get, list, SSE. | — | `ReportView`, `RequestQueue`, `ResearchReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ResearchRequest(String ticker, String question, String requestedBy) {}

record FundamentalsBundle(List<FundamentalFact> facts, Instant gatheredAt) {}
record FundamentalFact(String metric, String value, String period, String source) {}

record SentimentBundle(String overallSentiment, List<SentimentSignal> signals, Instant gatheredAt) {}
record SentimentSignal(String headline, String direction, String source) {}

record ResearchPlan(String fundamentalQuery, String sentimentQuery) {}

record SynthesisedReport(String summary, FundamentalsBundle fundamentals,
                         SentimentBundle sentiment, String sanitizerVerdict, Instant synthesisedAt) {}

record ResearchReport(
    String reportId,
    String ticker,
    String question,
    ReportStatus status,
    Optional<FundamentalsBundle> fundamentals,
    Optional<SentimentBundle> sentiment,
    Optional<SynthesisedReport> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { PLANNING, IN_PROGRESS, PUBLISHED, DEGRADED, BLOCKED }
```

### Events (on `ResearchReportEntity`)

`ReportCreated`, `FundamentalsAttached`, `SentimentAttached`, `ReportPublished`, `ReportDegraded`, `ReportBlocked`, `ReportEvalScored`.

### Events (on `RequestQueue`)

`ResearchSubmitted { reportId, ticker, question, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/reports` — body `{ ticker, question }` → `{ reportId }`. Starts a workflow.
- `GET /api/reports` — list all reports. Optional `?status=PLANNING|IN_PROGRESS|PUBLISHED|DEGRADED|BLOCKED`.
- `GET /api/reports/{id}` — one report.
- `GET /api/reports/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Investment Research Multi-Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a ticker and research question, live list of reports with status pills, expand-row to see fundamentals + sentiment + synthesised summary + eval score.

Browser title: `<title>Akka Sample: Investment Research Multi-Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — sector sanitizer** (`before-agent-response` on `ReportCoordinator`): vets the synthesised report for regulated investment content — prohibited price predictions, unattributed earnings forecasts, and jurisdiction-specific investment advice violations. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one published report every 5 minutes and emits a `ReportEvalScored` event with a 1–5 score and a short rationale assessing numeric-claim accuracy and source attribution.

## 9. Agent prompts

- `ReportCoordinator` → `prompts/report-coordinator.md`. Decomposes the research request into sub-tasks; later synthesises results into the final investment report.
- `EquityAnalyst` → `prompts/equity-analyst.md`. Gathers fundamental financial data; returns `FundamentalsBundle`.
- `MarketScout` → `prompts/market-scout.md`. Gathers market sentiment and price-action context; returns `SentimentBundle`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a ticker and question; report progresses PLANNING → IN_PROGRESS → PUBLISHED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `EquityAnalyst` timeout to 1 s); report enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a sanitizer failure (Coordinator returns regulated content); report enters BLOCKED.
4. **J4** — Wait after a successful publish; the report's row in the UI shows an eval score.
5. **J5** — Leave the service running; simulator drips requests; App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named investment-research-team demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-investment-research-team.
Java package io.akka.samples.investmentresearchmultiagent. Akka 3.6.0. HTTP port 9801.

Components to wire (exactly):
- 3 AutonomousAgents:
  * ReportCoordinator — definition() with capability(TaskAcceptance.of(PLAN_RESEARCH).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE_REPORT).maxIterationsPerTask(3)). System prompt loaded
    from prompts/report-coordinator.md. Returns ResearchPlan{fundamentalQuery, sentimentQuery} for
    PLAN_RESEARCH and SynthesisedReport{summary, fundamentals, sentiment, sanitizerVerdict, synthesisedAt}
    for SYNTHESISE_REPORT.
  * EquityAnalyst — capability(TaskAcceptance.of(GATHER_FUNDAMENTALS).maxIterationsPerTask(3)). System
    prompt from prompts/equity-analyst.md. Returns FundamentalsBundle{facts: List<FundamentalFact{metric,
    value, period, source}>, gatheredAt}.
  * MarketScout — capability(TaskAcceptance.of(GATHER_SENTIMENT).maxIterationsPerTask(2)). System prompt
    from prompts/market-scout.md. Returns SentimentBundle{overallSentiment, signals:
    List<SentimentSignal{headline, direction, source}>, gatheredAt}.

- 1 Workflow ReportWorkflow with steps:
  planStep -> [parallel] fundamentalsStep, sentimentStep -> joinStep -> synthesiseStep -> sanitizerStep -> emitStep.
  planStep calls forAutonomousAgent(ReportCoordinator.class, PLAN_RESEARCH).
  fundamentalsStep and sentimentStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(ReportWorkflow::fundamentalsStep, ofSeconds(60)) and
  stepTimeout(ReportWorkflow::sentimentStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with ReportDegraded.
  synthesiseStep calls forAutonomousAgent(ReportCoordinator.class, SYNTHESISE_REPORT) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. sanitizerStep runs the sector-scoped vetter plus
  LLM judge on the synthesised content; on failure, end with ReportBlocked. WorkflowSettings is
  nested inside Workflow — no import.

- 1 EventSourcedEntity ResearchReportEntity holding state ResearchReport{reportId, ticker, question,
  ReportStatus, Optional<FundamentalsBundle> fundamentals, Optional<SentimentBundle> sentiment,
  Optional<SynthesisedReport> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: PLANNING, IN_PROGRESS, PUBLISHED,
  DEGRADED, BLOCKED. Events: ReportCreated, FundamentalsAttached, SentimentAttached,
  ReportPublished, ReportDegraded, ReportBlocked, ReportEvalScored. Commands: createReport,
  attachFundamentals, attachSentiment, publishReport, degradeReport, blockReport,
  recordEval, getReport. emptyState() returns ResearchReport.initial("", "", null) with no
  commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueRequest(ticker, question, requestedBy) emitting
  ResearchSubmitted{reportId, ticker, question, requestedBy, submittedAt}.

- 1 View ReportView with row type ResearchReportRow (mirrors ResearchReport minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  ResearchReportEntity events. ONE query getAllReports SELECT * AS reports FROM report_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer RequestConsumer subscribed to RequestQueue events; on ResearchSubmitted
  starts a ReportWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-requests.jsonl and calls RequestQueue.enqueueRequest.
  * EvalSampler — every 5 minutes, queries ReportView.getAllReports, picks the oldest
    PUBLISHED report without an evalScore, runs a 1–5 rubric judge over the synthesised
    content assessing numeric-claim accuracy and source attribution, then calls
    ResearchReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * ReportEndpoint at /api with POST /reports, GET /reports, GET /reports/{id},
    GET /reports/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- ReportTasks.java declaring four Task<R> constants: PLAN_RESEARCH (ResearchPlan),
  GATHER_FUNDAMENTALS (FundamentalsBundle), GATHER_SENTIMENT (SentimentBundle),
  SYNTHESISE_REPORT (SynthesisedReport).
- Domain records ResearchPlan, FundamentalFact, FundamentalsBundle, SentimentSignal,
  SentimentBundle, SynthesisedReport.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9801 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-requests.jsonl with 8 canned request lines
  covering tickers such as NVDA, MSFT, TSLA, AMZN, AAPL, META, GOOGL, NFLX.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (S1 sector sanitizer
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector.primary = ["financial analysis",
  "investment research automation"], decisions.authority_level = recommend-only,
  data.pii = false, capabilities.* = false where not applicable; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/report-coordinator.md, prompts/equity-analyst.md, prompts/market-scout.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Investment Research Multi-Agent", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Investment Research Multi-Agent</title>. No
  subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (report-coordinator.json,
  equity-analyst.json, market-scout.json), picks one entry pseudo-randomly per
  call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    report-coordinator.json — list of either ResearchPlan or SynthesisedReport objects.
      4–6 ResearchPlan entries (fundamentalQuery + sentimentQuery pairs covering
      different tickers) and 4–6 SynthesisedReport entries (each with an 80–150 word
      summary, a 3–6 fundamental facts bundle, a 2–4 sentiment signal bundle,
      sanitizerVerdict = "ok").
    equity-analyst.json — 4–6 FundamentalsBundle entries, each with 3–5 facts
      whose metric values are plausible (e.g., P/E ratio, EPS, revenue growth,
      gross margin) and period values are quarterly or annual (e.g., "Q3 2024",
      "FY2024").
    market-scout.json — 4–6 SentimentBundle entries, each with overallSentiment
      in ["bullish","bearish","neutral"] and 2–4 signals whose direction values
      are one of "positive", "negative", "neutral".
- A MockModelProvider.seedFor(reportId) helper makes the selection
  deterministic per report id so the same report produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion ReportTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9801 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
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
