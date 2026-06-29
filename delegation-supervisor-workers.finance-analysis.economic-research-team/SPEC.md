# SPEC — economic-research-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Economic Research Agent.
**One-line pitch:** Submit a market question; a coordinator delegates data-gathering to a DataCollector and interpretation to an Economist in parallel, then synthesises one financial content-vetted analysis report.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a unified economic analysis report. The blueprint also demonstrates a **before-agent-response guardrail** that enforces financial content disclaimers before the report is returned to the reader.

## 3. User-facing flows

The user opens the App UI tab and submits a market question via the form.

1. The system creates an `AnalysisReport` record in `FRAMING` and starts an `AnalysisWorkflow`.
2. The Coordinator decomposes the question into two parallel work items: a data query for the DataCollector and an interpretive question for the Economist.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `SynthesisedReport { summary, indicators, interpretation, disclaimerVerdict }`.
5. A before-agent-response guardrail vets the report for speculative investment claims and missing disclaimers; if it fails, the report moves to `BLOCKED`. Otherwise, the report moves to `PUBLISHED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to synthesise from whichever side returned, and the report enters `DEGRADED`.

A `RequestSimulator` (TimedAction) drips a sample market question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EconomicsCoordinator` | `AutonomousAgent` | Decomposes the market question, synthesises the merged report, triggers the output guardrail. | `AnalysisWorkflow` | returns typed result to workflow |
| `DataCollector` | `AutonomousAgent` | Gathers quantitative indicators for the question. Seeded "market-data tool" returns canned results. | `AnalysisWorkflow` | — |
| `Economist` | `AutonomousAgent` | Interprets the indicators and proposes economic implications. | `AnalysisWorkflow` | — |
| `AnalysisWorkflow` | `Workflow` | Coordinates the parallel fan-out, synthesis, and guardrail. | `AnalysisEndpoint`, `MarketRequestConsumer` | `AnalysisReportEntity` |
| `AnalysisReportEntity` | `EventSourcedEntity` | Holds the report's lifecycle (framing → in-progress → published / degraded / blocked). | `AnalysisWorkflow` | `AnalysisView` |
| `RequestQueue` | `EventSourcedEntity` | Logs each submitted question for replay/audit. | `AnalysisEndpoint`, `RequestSimulator` | `MarketRequestConsumer` |
| `AnalysisView` | `View` | List-of-reports read model. | `AnalysisReportEntity` events | `AnalysisEndpoint` |
| `MarketRequestConsumer` | `Consumer` | Listens to `RequestQueue` events and starts a workflow per submission. | `RequestQueue` events | `AnalysisWorkflow` |
| `RequestSimulator` | `TimedAction` | Drips a sample market question every 60 s. | scheduler | `RequestQueue` |
| `EvalSampler` | `TimedAction` | Samples one published report every 5 minutes for eval scoring; emits a `ReportEvalScored` event. | scheduler | `AnalysisReportEntity` |
| `AnalysisEndpoint` | `HttpEndpoint` | `/api/analysis/*` — submit, get, list, SSE. | — | `AnalysisView`, `RequestQueue`, `AnalysisReportEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record MarketQuestion(String question, String requestedBy) {}

record IndicatorBundle(List<Indicator> indicators, Instant collectedAt) {}
record Indicator(String name, String value, String unit, String source) {}

record EconomicInterpretation(String thesis, List<String> implications, Instant interpretedAt) {}

record ResearchPlan(String dataQuery, String interpretiveQuestion) {}

record SynthesisedReport(String summary, IndicatorBundle indicators,
                         EconomicInterpretation interpretation,
                         String disclaimerVerdict, Instant synthesisedAt) {}

record AnalysisReport(
    String reportId,
    String question,
    ReportStatus status,
    Optional<IndicatorBundle> indicators,
    Optional<EconomicInterpretation> interpretation,
    Optional<SynthesisedReport> synthesised,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ReportStatus { FRAMING, IN_PROGRESS, PUBLISHED, DEGRADED, BLOCKED }
```

### Events (on `AnalysisReportEntity`)

`ReportCreated`, `IndicatorsAttached`, `InterpretationAttached`, `ReportPublished`, `ReportDegraded`, `ReportBlocked`, `ReportEvalScored`.

### Events (on `RequestQueue`)

`QuestionSubmitted { reportId, question, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/analysis` — body `{ question }` → `{ reportId }`. Starts a workflow.
- `GET /api/analysis` — list all reports. Optional `?status=FRAMING|IN_PROGRESS|PUBLISHED|DEGRADED|BLOCKED`.
- `GET /api/analysis/{id}` — one report.
- `GET /api/analysis/sse` — server-sent events stream of every report change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Economic Research Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a market question, live list of reports with status pills, expand-row to see indicators + interpretation + synthesised summary + eval score.

Browser title: `<title>Akka Sample: Economic Research Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `EconomicsCoordinator`): checks the synthesised report for speculative investment claims and missing financial disclaimers. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one published report every 5 minutes and emits a `ReportEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `EconomicsCoordinator` → `prompts/economics-coordinator.md`. Frames the market question into work items; later synthesises results into the final report.
- `DataCollector` → `prompts/data-collector.md`. Gathers quantitative indicators; returns `IndicatorBundle`.
- `Economist` → `prompts/economist.md`. Interprets indicators; returns `EconomicInterpretation`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a market question; report progresses FRAMING → IN_PROGRESS → PUBLISHED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `DataCollector` timeout to 1 s); report enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a guardrail failure (Coordinator returns a speculative investment claim without disclaimer); report enters BLOCKED.
4. **J4** — Wait after a successful publication; the report's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named economic-research-agent demonstrating the
delegation-supervisor-workers × finance-analysis cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-finance-analysis-economic-research-team.
Java package io.akka.samples.economicresearchagent. Akka 3.6.0. HTTP port 9263.

Components to wire (exactly):
- 3 AutonomousAgents:
  * EconomicsCoordinator — definition() with capability(TaskAcceptance.of(FRAME).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/economics-coordinator.md. Returns ResearchPlan{dataQuery, interpretiveQuestion}
    for FRAME and SynthesisedReport{summary, indicators, interpretation, disclaimerVerdict,
    synthesisedAt} for SYNTHESISE.
  * DataCollector — capability(TaskAcceptance.of(COLLECT).maxIterationsPerTask(3)). System prompt
    from prompts/data-collector.md. Returns IndicatorBundle{indicators: List<Indicator{name, value,
    unit, source}>, collectedAt}.
  * Economist — capability(TaskAcceptance.of(INTERPRET).maxIterationsPerTask(2)). System prompt from
    prompts/economist.md. Returns EconomicInterpretation{thesis, implications: List<String>,
    interpretedAt}.

- 1 Workflow AnalysisWorkflow with steps:
  frameStep -> [parallel] collectStep, interpretStep -> joinStep -> synthesiseStep ->
  guardrailStep -> emitStep.
  frameStep calls forAutonomousAgent(EconomicsCoordinator.class, FRAME).
  collectStep and interpretStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(AnalysisWorkflow::collectStep, ofSeconds(60)) and
  stepTimeout(AnalysisWorkflow::interpretStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with ReportDegraded.
  synthesiseStep calls forAutonomousAgent(EconomicsCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic vetter +
  LLM judge on the synthesised content; on failure, end with ReportBlocked. WorkflowSettings is
  nested inside Workflow — no import.

- 1 EventSourcedEntity AnalysisReportEntity holding state AnalysisReport{reportId, question,
  ReportStatus, Optional<IndicatorBundle> indicators, Optional<EconomicInterpretation>
  interpretation, Optional<SynthesisedReport> synthesised, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. ReportStatus enum: FRAMING, IN_PROGRESS, PUBLISHED, DEGRADED,
  BLOCKED. Events: ReportCreated, IndicatorsAttached, InterpretationAttached, ReportPublished,
  ReportDegraded, ReportBlocked, ReportEvalScored. Commands: createReport, attachIndicators,
  attachInterpretation, publishReport, degrade, block, recordEval, getReport.
  emptyState() returns AnalysisReport.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity RequestQueue with command enqueueQuestion(question, requestedBy) emitting
  QuestionSubmitted{reportId, question, requestedBy, submittedAt}.

- 1 View AnalysisView with row type AnalysisReportRow (mirrors AnalysisReport minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  AnalysisReportEntity events. ONE query getAllReports SELECT * AS reports FROM analysis_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer MarketRequestConsumer subscribed to RequestQueue events; on QuestionSubmitted
  starts an AnalysisWorkflow with the reportId as the workflow id.

- 2 TimedActions:
  * RequestSimulator — every 60s, reads next line from
    src/main/resources/sample-events/market-questions.jsonl and calls RequestQueue.enqueueQuestion.
  * EvalSampler — every 5 minutes, queries AnalysisView.getAllReports, picks the oldest
    PUBLISHED report without an evalScore, runs a 1–5 rubric judge over the synthesised
    content, then calls AnalysisReportEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * AnalysisEndpoint at /api with POST /analysis, GET /analysis, GET /analysis/{id},
    GET /analysis/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- EconomicsTasks.java declaring four Task<R> constants: FRAME (ResearchPlan), COLLECT
  (IndicatorBundle), INTERPRET (EconomicInterpretation), SYNTHESISE (SynthesisedReport).
- Domain records ResearchPlan, Indicator, IndicatorBundle, EconomicInterpretation, SynthesisedReport.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9263 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/market-questions.jsonl with 8 canned market-question lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = finance-analysis,
  decisions.authority_level = recommend-only, data.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/economics-coordinator.md, prompts/data-collector.md, prompts/economist.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Economic Research Agent", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Economic Research Agent</title>. No
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
  src/main/resources/mock-responses/<agent-name>.json (economics-coordinator.json,
  data-collector.json, economist.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    economics-coordinator.json — list of either ResearchPlan or SynthesisedReport objects.
      4–6 ResearchPlan entries (dataQuery + interpretiveQuestion pairs) and
      4–6 SynthesisedReport entries (each with an 80–150 word summary, a 3–6
      indicator bundle, a 3–6 implication interpretation, disclaimerVerdict = "ok").
    data-collector.json — 4–6 IndicatorBundle entries, each with 3–6 indicators
      whose source values are believable (e.g., "World Bank Open Data 2024",
      "IMF World Economic Outlook April 2024", "FRED Federal Reserve Economic Data").
    economist.json — 4–6 EconomicInterpretation entries, each with a one-sentence
      thesis taking an economic position and 3–6 short implication bullets.
- A MockModelProvider.seedFor(reportId) helper makes the selection
  deterministic per report id so the same report produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion EconomicsTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9263 in application.conf.
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
