# SPEC — fomc-event-analyst

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** FOMC Event Analyst.
**One-line pitch:** A user submits an FOMC meeting event; one `FomcAnalystAgent` walks it through three task phases — **GATHER** market data, **INTERPRET** policy signals into themes and rate-move implications, **SYNTHESIZE** a structured policy analysis — with a `before-agent-response` guardrail that reviews the final output for financial quality before it is recorded.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a finance-analysis domain. One `FomcAnalystAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the GATHER task's typed output (`MarketSnapshot`) becomes the INTERPRET task's instruction context; the INTERPRET task's typed output (`PolicySignalSet`) becomes the SYNTHESIZE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the recorded output of the SYNTHESIZE task. Before `AnalysisSynthesized` is written onto `FomcEventEntity`, `FinancialOutputGuardrail` inspects the candidate `PolicyAnalysis` for three financial-quality criteria: every cited rate-move rationale must trace to a `PolicySignal` from the recorded `PolicySignalSet`; every market-impact claim must reference a `MarketIndicator` from the recorded `MarketSnapshot`; and the analysis must not be empty. A candidate that fails any criterion is returned to the agent with a structured rejection so the SYNTHESIZE task loop can self-correct within its iteration budget. On rejection, a `GuardrailRejected` audit event is appended to the entity for full traceability.

The blueprint shows that guardrails are not limited to tool calls — a `before-agent-response` hook is the right cut when the contract to enforce is about the quality of the agent's final typed output, not about which tools it used to produce it.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types an **FOMC event identifier** into the input (or picks one of three seeded events — `2026-Q2 rate decision`, `2025-Q4 balance sheet runoff`, `2026-Q1 dot-plot release`).
2. The user clicks **Run analysis**. The UI POSTs to `/api/analyses` and receives an `analysisId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `GATHERING` — the workflow has started `gatherStep` and the agent has been handed the GATHER task.
4. Within ~10–20 s the card reaches `INTERPRETING` — the typed `MarketSnapshot` is visible in the card detail (a table of indicators with value, unit, and observation date). The agent's GATHER task returned; the workflow recorded `MarketDataGathered` and ran the INTERPRET task.
5. Within ~10–20 s more the card reaches `SYNTHESIZING`. The `PolicySignalSet` is visible (signal list with direction, magnitude, and grounding indicator).
6. Within ~10–20 s more the card reaches `REVIEWED`. The right pane now shows the full typed `PolicyAnalysis` — event name, rate outlook, per-theme `PolicySection` with body paragraphs and a `groundedIn` list — plus a guardrail review badge (ACCEPTED / REJECTED-AND-RETRIED) and a one-line rationale.
7. The user can submit another event; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `FomcEndpoint` | `HttpEndpoint` | `/api/analyses/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `FomcEventEntity`, `PolicyAnalysisView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `FomcEventEntity` | `EventSourcedEntity` | Per-analysis lifecycle: created → gathering → gathered → interpreting → interpreted → synthesizing → synthesized → reviewed. Source of truth. | `FomcEndpoint`, `FomcPipelineWorkflow` | `PolicyAnalysisView` |
| `FomcPipelineWorkflow` | `Workflow` | One workflow per analysisId. Steps: `gatherStep` → `interpretStep` → `synthesizeStep` → `reviewStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `FomcEndpoint` after `CREATED` | `FomcAnalystAgent`, `FomcEventEntity` |
| `FomcAnalystAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `FomcTasks.java`: `GATHER_MARKET_DATA` → `MarketSnapshot`, `INTERPRET_POLICY_SIGNALS` → `PolicySignalSet`, `SYNTHESIZE_ANALYSIS` → `PolicyAnalysis`. Each task is registered with the phase-appropriate function tools. | invoked by `FomcPipelineWorkflow` | returns typed results |
| `GatherTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchIndicators(eventId)` and `fetchYieldCurve(eventId)`. Reads from `src/main/resources/sample-data/market/*.json` for deterministic offline output. | called from GATHER task | returns `List<MarketIndicator>` / `YieldCurveSnapshot` |
| `InterpretTools` | function-tools class | Implements `extractPolicySignals(snapshot)` and `classifyRateMove(signals)`. Pure in-memory transformations. | called from INTERPRET task | returns `List<PolicySignal>` / `RateMoveForecast` |
| `SynthesizeTools` | function-tools class | Implements `draftPolicySection(signal, indicators)` and `compileGrounding(signals)`. | called from SYNTHESIZE task | returns `PolicySection` / `List<GroundingRef>` |
| `FinancialOutputGuardrail` | `before-agent-response` guardrail (registered on `FomcAnalystAgent`) | Inspects the candidate `PolicyAnalysis` returned by the SYNTHESIZE task. Checks rate-move attribution, market-impact grounding, and non-emptiness. Rejects non-compliant responses with a structured reason. | every SYNTHESIZE task final response | accept / structured-reject |
| `PolicyAnalysisView` | `View` | Read model: one row per analysis for the UI. | `FomcEventEntity` events | `FomcEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MarketIndicator(
    String indicatorId,
    String name,
    double value,
    String unit,
    Instant observedAt
) {}

record YieldCurveSnapshot(
    double twoYear,
    double tenYear,
    double spread,
    Instant snapshotAt
) {}

record MarketSnapshot(
    String eventId,
    List<MarketIndicator> indicators,
    YieldCurveSnapshot yieldCurve,
    Instant gatheredAt
) {}

record PolicySignal(
    String signalId,
    String direction,        // "hawkish" | "dovish" | "neutral"
    String magnitude,        // "strong" | "moderate" | "weak"
    String rationale,
    String groundedIndicatorId   // must equal a MarketIndicator.indicatorId
) {}

record RateMoveForecast(
    String forecastId,
    int basisPoints,         // expected move in bps; 0 = hold
    double confidence,       // 0.0–1.0
    String narrative
) {}

record PolicySignalSet(
    List<PolicySignal> signals,
    RateMoveForecast forecast,
    Instant interpretedAt
) {}

record GroundingRef(
    String indicatorId,
    String indicatorName,
    double value,
    String unit
) {}

record PolicySection(
    String signalId,
    String heading,
    String body,
    List<GroundingRef> groundedIn
) {}

record PolicyAnalysis(
    String eventName,
    String rateOutlook,
    String executiveSummary,
    List<PolicySection> sections,
    RateMoveForecast forecast,
    Instant synthesizedAt
) {}

record ReviewOutcome(
    String verdict,          // "ACCEPTED" | "REJECTED"
    String reason,
    Instant reviewedAt
) {}

record AnalysisRecord(
    String analysisId,
    Optional<String> eventId,
    Optional<MarketSnapshot> snapshot,
    Optional<PolicySignalSet> signalSet,
    Optional<PolicyAnalysis> analysis,
    Optional<ReviewOutcome> review,
    AnalysisStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum AnalysisStatus {
    CREATED, GATHERING, GATHERED, INTERPRETING, INTERPRETED,
    SYNTHESIZING, SYNTHESIZED, REVIEWED, FAILED
}
```

Events on `FomcEventEntity`: `AnalysisCreated`, `GatherStarted`, `MarketDataGathered`, `InterpretStarted`, `PolicySignalsInterpreted`, `SynthesizeStarted`, `AnalysisSynthesized`, `OutputReviewed`, `GuardrailRejected`, `AnalysisFailed`.

Every nullable lifecycle field on the `AnalysisRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/analyses` — body `{ eventId }` → `{ analysisId }`.
- `GET /api/analyses` — list all analyses, newest-first.
- `GET /api/analyses/{id}` — one analysis.
- `GET /api/analyses/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: FOMC Event Analyst</title>`.

The App UI tab is a two-column layout: a left rail with the live list of analyses (status pill + event name + age + guardrail verdict badge) and a right pane with the selected analysis's detail — event name, market snapshot indicators table, policy signals list, policy analysis sections, forecast chip, and a guardrail review log strip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (financial output review)**: `FinancialOutputGuardrail` is registered on `FomcAnalystAgent` and runs before the SYNTHESIZE task's typed result is accepted. It inspects the candidate `PolicyAnalysis` for three quality criteria: (1) rate-move attribution — every `PolicySection.signalId` must equal a `PolicySignal.signalId` from the recorded `PolicySignalSet`; (2) market-impact grounding — every `PolicySection.groundedIn[i].indicatorId` must equal a `MarketIndicator.indicatorId` from the recorded `MarketSnapshot`; (3) non-emptiness — `sections` must contain at least one entry. A candidate that fails any criterion is returned to the agent loop with a structured `financial-quality-violation` error naming the first failing check; the workflow records a `GuardrailRejected{phase, reason}` event. The agent loop retries within its 4-iteration budget. On accept, the guardrail records a `ReviewOutcome{verdict: "ACCEPTED"}` event.

## 9. Agent prompts

- `FomcAnalystAgent` → `prompts/fomc-analyst-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools appropriate to that phase, treat each task's typed input as the entire context for that phase, and return the task's typed output grounded in the inputs.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded event `2026-Q2 rate decision`; within 60 s the analysis reaches `REVIEWED` with non-empty indicators, ≥ 2 policy signals, ≥ 2 sections, and an ACCEPTED guardrail badge.
2. **J2** — The agent's SYNTHESIZE task produces a `PolicySection` whose `groundedIn` references an indicator absent from the `MarketSnapshot` (mock LLM path). `FinancialOutputGuardrail` rejects the response; a `GuardrailRejected` event lands on the entity; the agent retries; the analysis eventually completes with an ACCEPTED badge. The UI's review-log strip shows the one rejection.
3. **J3** — Each task's instructions and tool calls are visible in the per-analysis trace (logged at `INFO`); the GATHER task's log shows only GATHER-tool calls, the INTERPRET task's log shows only INTERPRET-tool calls, the SYNTHESIZE task's log shows only SYNTHESIZE-tool calls.
4. **J4** — A SYNTHESIZE task that returns an empty `sections` list is rejected by the guardrail with a `non-empty-sections` violation; the card shows `REJECTED-AND-RETRIED` in the review-log strip until the agent self-corrects.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named fomc-event-analyst demonstrating the sequential-pipeline x
finance-analysis cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact sequential-pipeline-finance-analysis-fomc-event-analyst.
Java package io.akka.samples.fomcresearch. Akka 3.6.0. HTTP port 9498.

Components to wire (exactly):

- 1 AutonomousAgent FomcAnalystAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/fomc-analyst-agent.md>) and three .capability(TaskAcceptance.of(TASK)
  .maxIterationsPerTask(4)) entries — one per declared Task. Function tools are registered
  with .tools(...) — the GATHER, INTERPRET, and SYNTHESIZE tool sets are ALL registered on
  the agent. The before-agent-response guardrail (FinancialOutputGuardrail) is registered on
  the agent via the agent's guardrail-configuration block. On guardrail rejection the agent
  loop retries within its 4-iteration budget.

- 1 Workflow FomcPipelineWorkflow per analysisId with four steps:
  * gatherStep — emits GatherStarted on the entity, then calls componentClient
    .forAutonomousAgent(FomcAnalystAgent.class, "agent-" + analysisId).runSingleTask(
      TaskDef.instructions("Event: " + eventId + "\nPhase: GATHER\nUse fetchIndicators
      and fetchYieldCurve to collect market data for this FOMC event.")
        .metadata("analysisId", analysisId)
        .metadata("phase", "GATHER")
        .taskType(FomcTasks.GATHER_MARKET_DATA)
    ). Reads forTask(taskId).result(GATHER_MARKET_DATA) to get MarketSnapshot. Writes
    FomcEventEntity.recordSnapshot(snapshot). WorkflowSettings.stepTimeout 60s.
  * interpretStep — emits InterpretStarted, then runSingleTask with TaskDef.instructions
    (formatInterpretContext(snapshot, eventId)) and metadata.phase = "INTERPRET", taskType
    INTERPRET_POLICY_SIGNALS. Writes FomcEventEntity.recordSignalSet(signalSet). stepTimeout 60s.
  * synthesizeStep — emits SynthesizeStarted, then runSingleTask with TaskDef.instructions
    (formatSynthesizeContext(signalSet, snapshot, eventId)) and metadata.phase = "SYNTHESIZE",
    taskType SYNTHESIZE_ANALYSIS. The before-agent-response guardrail fires before the typed
    result is accepted; on acceptance writes FomcEventEntity.recordAnalysis(analysis) and
    FomcEventEntity.recordReview(ReviewOutcome.ACCEPTED). stepTimeout 60s.
  * reviewStep — runs deterministic audit: verifies the recorded review outcome and writes
    the final OutputReviewed event. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(FomcPipelineWorkflow::error). The error step writes
  AnalysisFailed and ends.

- 1 EventSourcedEntity FomcEventEntity (one per analysisId). State AnalysisRecord{analysisId,
  eventId: Optional<String>, snapshot: Optional<MarketSnapshot>, signalSet:
  Optional<PolicySignalSet>, analysis: Optional<PolicyAnalysis>, review:
  Optional<ReviewOutcome>, status: AnalysisStatus, createdAt: Instant, finishedAt:
  Optional<Instant>}. AnalysisStatus enum: CREATED, GATHERING, GATHERED, INTERPRETING,
  INTERPRETED, SYNTHESIZING, SYNTHESIZED, REVIEWED, FAILED. Events: AnalysisCreated{eventId},
  GatherStarted, MarketDataGathered{snapshot}, InterpretStarted, PolicySignalsInterpreted
  {signalSet}, SynthesizeStarted, AnalysisSynthesized{analysis}, OutputReviewed{review},
  GuardrailRejected{phase, reason}, AnalysisFailed{reason}. Commands: create, startGather,
  recordSnapshot, startInterpret, recordSignalSet, startSynthesize, recordAnalysis,
  recordReview, recordGuardrailRejection, fail, getAnalysis. emptyState() returns
  AnalysisRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3).

- 1 View PolicyAnalysisView with row type AnalysisRow that mirrors AnalysisRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes FomcEventEntity events.
  ONE query getAllAnalyses: SELECT * AS analyses FROM analysis_view. No WHERE status filter
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * FomcEndpoint at /api with POST /analyses (body {eventId}; mints analysisId; calls
    FomcEventEntity.create(eventId); then starts FomcPipelineWorkflow with id
    "pipeline-" + analysisId; returns {analysisId}), GET /analyses (list from
    getAllAnalyses, sorted newest-first), GET /analyses/{id} (one row), GET
    /analyses/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- FomcTasks.java declaring three Task<R> constants:
    GATHER_MARKET_DATA = Task.name("Gather market data").description("Collect market
      indicators and yield curve snapshot for an FOMC event using fetchIndicators and
      fetchYieldCurve").resultConformsTo(MarketSnapshot.class);
    INTERPRET_POLICY_SIGNALS = Task.name("Interpret policy signals").description("Extract
      policy signals from a MarketSnapshot, then classify the rate-move direction and
      magnitude").resultConformsTo(PolicySignalSet.class);
    SYNTHESIZE_ANALYSIS = Task.name("Synthesize analysis").description("Compose a
      PolicyAnalysis whose PolicySections mirror the policy signals one-to-one, grounded in
      the recorded market indicators").resultConformsTo(PolicyAnalysis.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- GatherTools.java — @FunctionTool fetchIndicators(String eventId) -> List<MarketIndicator>
  reading from src/main/resources/sample-data/market/<eventId>.json; @FunctionTool
  fetchYieldCurve(String eventId) -> YieldCurveSnapshot reading the yieldCurve node of
  the same file.

- InterpretTools.java — @FunctionTool extractPolicySignals(MarketSnapshot snapshot) ->
  List<PolicySignal> (one PolicySignal per MarketIndicator, with signalId minted as
  "ps-" + sha1(indicatorId).substring(0,8)); @FunctionTool classifyRateMove(List<PolicySignal>
  signals) -> RateMoveForecast (deterministic rule: if majority of signals are "hawkish" →
  basisPoints = 25, "dovish" → basisPoints = -25, else 0; confidence from signal count).

- SynthesizeTools.java — @FunctionTool draftPolicySection(PolicySignal signal, List<
  MarketIndicator> indicators) -> PolicySection (heading from signal direction + magnitude;
  body paraphrases the rationale; groundedIn contains the matching MarketIndicator);
  @FunctionTool compileGrounding(List<PolicySignal> signals) -> List<GroundingRef> (one
  GroundingRef per distinct groundedIndicatorId, label and value from the MarketSnapshot).

- FinancialOutputGuardrail.java — implements the before-agent-response hook. On every
  SYNTHESIZE task final candidate: (1) check every PolicySection.signalId exists in the
  recorded PolicySignalSet.signals; (2) check every groundedIn.indicatorId exists in the
  recorded MarketSnapshot.indicators; (3) check sections is non-empty. On any failure
  returns Guardrail.reject("financial-quality-violation: <first-failing-check>"). On reject
  ALSO calls FomcEventEntity.recordGuardrailRejection(phase, reason) so the rejection is
  visible in the UI's review-log strip and in the audit log. On accept, records
  ReviewOutcome{verdict: "ACCEPTED"}.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9498 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/events.jsonl with 5 seeded event lines covering the three
  surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/market/*.json — three files keyed by seeded eventId, each
  carrying 5-8 MarketIndicator entries plus a YieldCurveSnapshot with deterministic content
  so GatherTools.fetchIndicators returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (events are
  market-level, not person-level), decisions.authority_level = recommend-only (the analysis
  is advisory), oversight.human_in_loop = true (an analyst reviews before acting),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "ungrounded-rate-attribution", "empty-sections",
  "guardrail-rejection-loop", "stale-market-data"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/fomc-analyst-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: FOMC Event Analyst", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of analysis cards; right = selected-analysis detail with event header, market
  indicators table, policy signals list, analysis sections, forecast chip, review badge,
  review-log strip). Browser title exactly: <title>Akka Sample: FOMC Event Analyst</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(analysisId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences.
- Per-task mock-response shapes for THIS blueprint:
    gather-market-data.json — 5 MarketSnapshot entries, each with 5-8 MarketIndicator
      items and a YieldCurveSnapshot per seeded event.
    interpret-policy-signals.json — 5 PolicySignalSet entries paired one-to-one with the
      gather entries, each with 2-4 PolicySignal items and a RateMoveForecast.
    synthesize-analysis.json — 5 PolicyAnalysis entries paired one-to-one. Each carries
      2-4 PolicySection items matching the paired PolicySignalSet signals, groundedIn
      references matching the paired MarketSnapshot indicators. Plus 1 UNGROUNDED entry
      whose first PolicySection.groundedIn references an indicatorId absent from the paired
      MarketSnapshot — the guardrail rejects it and the mock then falls through to a valid
      entry. The ungrounded entry is selected on the FIRST iteration of every 3rd analysis
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(analysisId) helper makes per-analysis selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FomcAnalystAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion FomcTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (gatherStep
  60s, interpretStep 60s, synthesizeStep 60s, reviewStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the AnalysisRecord row record is Optional<T>.
- Lesson 7: FomcTasks.java with GATHER_MARKET_DATA, INTERPRET_POLICY_SIGNALS,
  SYNTHESIZE_ANALYSIS constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9498 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (FomcAnalystAgent).
- The sequential-pipeline invariant: each phase's tool set is registered on the agent;
  the before-agent-response guardrail (FinancialOutputGuardrail) reviews the SYNTHESIZE
  task's final output. Do NOT conditionally register tools per task.
- Task dependency is carried by typed task results: gatherStep writes MarketSnapshot onto
  the entity, interpretStep reads it and builds the INTERPRET task's instruction context
  from it, synthesizeStep reads both. The agent itself is stateless across phases.
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
