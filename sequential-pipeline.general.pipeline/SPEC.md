# SPEC — pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Pipeline Briefing.
**One-line pitch:** A user submits a topic; one `ReportAgent` walks it through three task phases — **COLLECT** raw signals, **ANALYZE** them into themes and claims, **WRITE** a structured briefing — with each phase gated on the prior phase's recorded output and each phase's tools rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a general-purpose briefing domain. One `ReportAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the COLLECT task's typed output becomes the ANALYZE task's instruction context; the ANALYZE task's typed output becomes the WRITE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`COLLECT` / `ANALYZE` / `REPORT`) and the current `BriefingEntity` status. A REPORT-phase tool called while the entity has not yet recorded `AnalysisProduced` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. The same hook enforces the deeper pipeline property: a phase's tool set is unreachable until its preconditions hold.
- An **`on-decision-eval`** runs immediately after `ReportWritten` lands, as `evalStep` inside the workflow. A deterministic, rule-based `CompletenessScorer` (no LLM call — the eval is rule-based on purpose, so the same briefing always scores the same) checks that every analysis theme has a matching report section, that every report section cites at least one signal source, and that no signal source referenced in a section was absent from the collected `SignalSet`.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and the per-phase tool isolation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **topic** into the input (or picks one of three seeded topics — `AI regulation Q3 2026`, `LLM token-cost trends`, `Postgres release 17 retrospective`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/briefings` and receives a `briefingId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `COLLECTING` — the workflow has started `collectStep` and the agent has been handed the COLLECT task.
4. Within ~10–20 s the card reaches `ANALYZING` — the typed `SignalSet` is visible in the card detail (a small table of signals with source and snippet preview). The agent's COLLECT task returned; the workflow recorded `SignalsCollected` and ran the ANALYZE task.
5. Within ~10–20 s more the card reaches `REPORTING`. The `Analysis` is visible (theme list + claim list with theme assignments).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `Briefing` — title, summary, per-theme `Section` with body paragraphs and a `sources` list — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another topic; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BriefingEndpoint` | `HttpEndpoint` | `/api/briefings/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `BriefingEntity`, `BriefingView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BriefingEntity` | `EventSourcedEntity` | Per-briefing lifecycle: created → collecting → collected → analyzing → analyzed → reporting → reported → evaluated. Source of truth. | `BriefingEndpoint`, `BriefingPipelineWorkflow` | `BriefingView` |
| `BriefingPipelineWorkflow` | `Workflow` | One workflow per briefing. Steps: `collectStep` → `analyzeStep` → `reportStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `BriefingEndpoint` after `CREATED` | `ReportAgent`, `BriefingEntity` |
| `ReportAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `ReportTasks.java`: `COLLECT_SIGNALS` → `SignalSet`, `ANALYZE_SIGNALS` → `Analysis`, `WRITE_REPORT` → `Briefing`. Each task is registered with the phase-appropriate function tools. | invoked by `BriefingPipelineWorkflow` | returns typed results |
| `CollectTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchSignals(topic)` and `fetchSnippet(url)`. Reads from `src/main/resources/sample-data/signals/*.json` for deterministic offline output. | called from COLLECT task | returns `List<Signal>` |
| `AnalyzeTools` | function-tools class | Implements `extractClaims(signals)` and `clusterClaims(claims)`. Pure in-memory transformations. | called from ANALYZE task | returns `List<Claim>` / `List<Theme>` |
| `ReportTools` | function-tools class | Implements `formatSection(theme, claims)` and `gatherSources(claims)`. | called from REPORT task | returns `Section` / `List<Source>` |
| `PhaseGuardrail` | `before-tool-call` guardrail (registered on `ReportAgent`) | Reads the in-flight task's declared phase and the current `BriefingEntity` status. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `CompletenessScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `Briefing`, `Analysis`, `SignalSet`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `BriefingView` | `View` | Read model: one row per briefing for the UI. | `BriefingEntity` events | `BriefingEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Signal(String source, String url, String snippet, Instant capturedAt) {}

record SignalSet(List<Signal> signals, Instant collectedAt) {}

record Claim(String claimId, String text, String supportingSource) {}

record Theme(String themeId, String label, List<String> claimIds) {}

record Analysis(List<Theme> themes, List<Claim> claims, Instant analyzedAt) {}

record Source(String label, String url) {}

record Section(String themeId, String heading, String body, List<Source> sources) {}

record Briefing(
    String title,
    String summary,
    List<Section> sections,
    Instant writtenAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record BriefingRecord(
    String briefingId,
    Optional<String> topic,
    Optional<SignalSet> signals,
    Optional<Analysis> analysis,
    Optional<Briefing> briefing,
    Optional<EvalResult> eval,
    BriefingStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum BriefingStatus {
    CREATED, COLLECTING, COLLECTED, ANALYZING, ANALYZED,
    REPORTING, REPORTED, EVALUATED, FAILED
}
```

Events on `BriefingEntity`: `BriefingCreated`, `CollectStarted`, `SignalsCollected`, `AnalyzeStarted`, `AnalysisProduced`, `ReportStarted`, `ReportWritten`, `EvaluationScored`, `GuardrailRejected`, `BriefingFailed`.

Every nullable lifecycle field on the `BriefingRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/briefings` — body `{ topic }` → `{ briefingId }`.
- `GET /api/briefings` — list all briefings, newest-first.
- `GET /api/briefings/{id}` — one briefing.
- `GET /api/briefings/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Pipeline Briefing</title>`.

The App UI tab is a two-column layout: a left rail with the live list of briefings (status pill + topic + age) and a right pane with the selected briefing's detail — topic, collected signals table, analysis themes/claims, briefing sections, eval score chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (phase-gate)**: `PhaseGuardrail` is registered on `ReportAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.COLLECT`, `Phase.ANALYZE`, `Phase.REPORT`) and the current `BriefingEntity.status` for the briefing the task is bound to. The accept rule is precise: `COLLECT` tools require `status ∈ {CREATED, COLLECTING}`; `ANALYZE` tools require `status ∈ {COLLECTED, ANALYZING}` AND `signals.isPresent()`; `REPORT` tools require `status ∈ {ANALYZED, REPORTING}` AND `analysis.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget.
- **E1 — `on-decision-eval`**: runs immediately after `ReportWritten` lands, as `evalStep` inside the workflow. `CompletenessScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every analysis theme must have a matching section (theme coverage), every section must cite at least one source (source attribution), every cited source URL must appear in the recorded `SignalSet` (no hallucinated sources), and the section count must equal the theme count (no silent expansion or collapse). Emits `EvaluationScored{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `ReportAgent` → `prompts/report-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded topic `LLM token-cost trends`; within 60 s the briefing reaches `EVALUATED` with non-empty signals, ≥ 2 themes, ≥ 2 sections, and an eval score chip on the card.
2. **J2** — The agent's first iteration on a briefing calls a REPORT-phase tool (`formatSection`) before `AnalysisProduced` has been recorded (mock LLM path). `PhaseGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the briefing eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A briefing whose mock-LLM trajectory produces a section citing a URL absent from the recorded `SignalSet` is scored 1 with a rationale naming the hallucinated source; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-briefing trace (logged at `INFO`); the COLLECT task's log shows only COLLECT-tool calls, the ANALYZE task's log shows only ANALYZE-tool calls, the REPORT task's log shows only REPORT-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named pipeline demonstrating the sequential-pipeline x general cell. Runs out
of the box (no external services). Maven group io.akka.samples. Maven artifact pipeline.
Java package io.akka.samples.pipeline. Akka 3.6.0. HTTP port 9652.

Components to wire (exactly):

- 1 AutonomousAgent ReportAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/report-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  COLLECT, ANALYZE, and REPORT tool sets are ALL registered on the agent; phase gating is the
  job of PhaseGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (PhaseGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow BriefingPipelineWorkflow per briefingId with four steps:
  * collectStep — emits CollectStarted on the entity, then calls componentClient
    .forAutonomousAgent(ReportAgent.class, "agent-" + briefingId).runSingleTask(
      TaskDef.instructions("Topic: " + topic + "\nPhase: COLLECT\nUse the search and fetch
      tools to collect 4-8 signals about this topic.")
        .metadata("briefingId", briefingId)
        .metadata("phase", "COLLECT")
        .taskType(ReportTasks.COLLECT_SIGNALS)
    ). Reads forTask(taskId).result(COLLECT_SIGNALS) to get SignalSet. Writes
    BriefingEntity.recordSignals(signalSet). WorkflowSettings.stepTimeout 60s.
  * analyzeStep — emits AnalyzeStarted, then runSingleTask with TaskDef.instructions
    (formatAnalyzeContext(signalSet, topic)) and metadata.phase = "ANALYZE", taskType
    ANALYZE_SIGNALS. Writes BriefingEntity.recordAnalysis(analysis). stepTimeout 60s.
  * reportStep — emits ReportStarted, then runSingleTask with TaskDef.instructions
    (formatReportContext(analysis, signalSet, topic)) and metadata.phase = "REPORT", taskType
    WRITE_REPORT. Writes BriefingEntity.recordReport(briefing). stepTimeout 60s.
  * evalStep — runs the deterministic CompletenessScorer over (briefing, analysis,
    signalSet) and writes BriefingEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(BriefingPipelineWorkflow::error). The error step writes
  BriefingFailed and ends.

- 1 EventSourcedEntity BriefingEntity (one per briefingId). State BriefingRecord{briefingId,
  topic: Optional<String>, signals: Optional<SignalSet>, analysis: Optional<Analysis>,
  briefing: Optional<Briefing>, eval: Optional<EvalResult>, status: BriefingStatus, createdAt:
  Instant, finishedAt: Optional<Instant>}. BriefingStatus enum: CREATED, COLLECTING,
  COLLECTED, ANALYZING, ANALYZED, REPORTING, REPORTED, EVALUATED, FAILED. Events:
  BriefingCreated{topic}, CollectStarted, SignalsCollected{signals}, AnalyzeStarted,
  AnalysisProduced{analysis}, ReportStarted, ReportWritten{briefing},
  EvaluationScored{eval}, GuardrailRejected{phase, tool, reason}, BriefingFailed{reason}.
  Commands: create, startCollect, recordSignals, startAnalyze, recordAnalysis, startReport,
  recordReport, recordEvaluation, recordGuardrailRejection, fail, getBriefing. emptyState()
  returns BriefingRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View BriefingView with row type BriefingRow that mirrors BriefingRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes BriefingEntity events. ONE
  query getAllBriefings: SELECT * AS briefings FROM briefing_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * BriefingEndpoint at /api with POST /briefings (body {topic}; mints briefingId; calls
    BriefingEntity.create(topic); then starts BriefingPipelineWorkflow with id
    "pipeline-" + briefingId; returns {briefingId}), GET /briefings (list from
    getAllBriefings, sorted newest-first), GET /briefings/{id} (one row), GET
    /briefings/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- ReportTasks.java declaring three Task<R> constants:
    COLLECT_SIGNALS = Task.name("Collect signals").description("Gather raw signals about a
      topic by calling searchSignals and fetchSnippet").resultConformsTo(SignalSet.class);
    ANALYZE_SIGNALS = Task.name("Analyze signals").description("Extract claims, then cluster
      claims into themes").resultConformsTo(Analysis.class);
    WRITE_REPORT = Task.name("Write report").description("Compose a Briefing whose Sections
      mirror the Analysis themes").resultConformsTo(Briefing.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {COLLECT, ANALYZE, REPORT}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "searchSignals", phase = Phase.COLLECT)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- CollectTools.java — @FunctionTool searchSignals(String topic) -> List<Signal> reading from
  src/main/resources/sample-data/signals/*.json keyed by topic; @FunctionTool fetchSnippet(
  String url) -> String reading from the matching signal entry's snippet.

- AnalyzeTools.java — @FunctionTool extractClaims(List<Signal>) -> List<Claim> (one Claim per
  Signal, with claimId minted as "c-" + sha1(snippet).substring(0,8)); @FunctionTool
  clusterClaims(List<Claim>) -> List<Theme> (deterministic clustering — assign each Claim to
  a Theme keyed by the first significant noun phrase of its text; 2-5 themes typical).

- ReportTools.java — @FunctionTool formatSection(Theme, List<Claim>) -> Section (heading
  from theme label, body composed from the claim texts joined with linking phrases); @
  FunctionTool gatherSources(List<Claim>) -> List<Source> (one Source per distinct
  supportingSource, label is the source name, url is the original signal url).

- PhaseGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the BriefingEntity status by briefingId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("phase-violation: <tool> requires <precondition>, saw
  <status>"). On reject ALSO calls BriefingEntity.recordGuardrailRejection(phase, tool,
  reason) so the rejection is visible in the UI's rejection-log strip and in the audit log.

- CompletenessScorer.java — pure deterministic logic (no LLM). Inputs: Briefing, Analysis,
  SignalSet. Outputs: EvalResult with score and rationale. Four checks, one point per check
  satisfied, starting from a base of 1: theme coverage (every Theme has a Section), source
  attribution (every Section has >= 1 Source), source provenance (every Source.url appears
  in SignalSet.signals[].url), and section parity (sections.size() == themes.size()). Score
  range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9652 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/topics.jsonl with 5 seeded topic lines covering the three
  surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/signals/*.json — three files keyed by seeded topic, each
  carrying 6-10 Signal entries with deterministic content so CollectTools.searchSignals
  returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (signals are
  topic-level, not person-level), decisions.authority_level = recommend-only (the briefing
  is advisory), oversight.human_in_loop = true (a human reads the briefing before acting on
  it), operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "hallucinated-source", "missing-theme-coverage",
  "phase-violation", "evidence-thin-section"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/report-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Pipeline Briefing", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of briefing cards; right = selected-briefing detail with topic header, signals
  table, analysis themes/claims, briefing sections, eval-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Pipeline Briefing</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(briefingId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    collect-signals.json — 6 SignalSet entries, each with 5-8 Signal items per seeded
      topic. Each entry's tool_calls array contains 2-4 calls: 1 searchSignals(topic) + 1-3
      fetchSnippet(url) calls. Plus 1 deliberately PHASE-VIOLATING entry whose tool_calls
      array starts with a formatSection(...) call (a REPORT-phase tool called during the
      COLLECT phase) — the guardrail rejects it, the mock then falls through to a normal
      collect sequence. The mock should select the violating entry on the FIRST iteration
      of every 3rd briefing (modulo seed) so J2 is reproducible.
    analyze-signals.json — 6 Analysis entries paired one-to-one with the collect entries,
      each with 2-4 Theme items and 5-8 Claim items, with tool_calls containing
      extractClaims + clusterClaims in order.
    write-report.json — 6 Briefing entries paired one-to-one. Each carries 2-4 Section
      items matching the paired Analysis themes, sources lists referencing the paired
      SignalSet urls, tool_calls containing formatSection (one per theme) + gatherSources.
      Plus 1 deliberately HALLUCINATED-SOURCE entry whose first Section cites a url absent
      from the paired SignalSet — the workflow's evalStep scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(briefingId) helper makes per-briefing selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ReportAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ReportTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (collectStep
  60s, analyzeStep 60s, reportStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the BriefingRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ReportTasks.java with COLLECT_SIGNALS, ANALYZE_SIGNALS, WRITE_REPORT constants
  is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9652 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ReportAgent). The
  on-decision eval is rule-based (CompletenessScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (PhaseGuardrail) is the runtime mechanism that enforces
  the phase order. Do NOT conditionally register tools per task — the guardrail is the
  gate.
- Task dependency is carried by typed task results: collectStep writes SignalSet onto the
  entity, analyzeStep reads it and builds the ANALYZE task's instruction context from it,
  reportStep reads both. The agent itself is stateless across phases.
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
