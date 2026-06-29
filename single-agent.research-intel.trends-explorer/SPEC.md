# SPEC — google-trends-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Google Trends Agent.
**One-line pitch:** A user submits a region and time window; one AI agent synthesises trending search topics from seeded data (passed as a task attachment, never as inline prompt text) and returns a structured `TrendReport` — ranked topics, breakout signals, and related query clusters.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intelligence domain. One `TrendsExplorerAgent` (AutonomousAgent) carries the entire synthesis; surrounding components prepare its input and audit its output. There are no governance controls in this baseline, which makes it the starting point a deployer customises before wiring their own.

The blueprint shows how the single-agent pattern wires a clean lifecycle — request intake, data assembly, LLM synthesis, result persistence, and an on-decision evaluator — without requiring any pre-existing external service. The seeded trend datasets live in-process; a real data-source integration is a single-component swap.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **Region** from a dropdown (US, EU, APAC, LATAM, Global) or enters a custom BCP-47 region code.
2. The user selects a **Time window** (Past day, Past week, Past month, Past 90 days).
3. The user picks an **Interest category** (All, Technology, Finance, Health, Entertainment) or enters a custom topic seed.
4. The user clicks **Fetch trends**. The UI POSTs to `/api/trends` and receives a `requestId`.
5. The card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `DATA_READY` — the raw trend payload is assembled from the seeded dataset for that region/window pair.
6. Within ~10–30 s, the workflow's `synthesizeStep` completes. The card transitions to `SYNTHESIZING` then `REPORT_READY`. The report appears: a ranked list of trending topics (position, name, search-volume index, breakout flag), a breakout signal summary, and a table of related query clusters (anchor term → top 3 related queries).
7. Within ~1 s of the report, the `evalStep` finishes. The card shows an **eval score** chip (1–5) plus a one-line rationale describing whether the report's topics are well-supported.
8. The user can submit another request; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `TrendEndpoint` | `HttpEndpoint` | `/api/trends/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `TrendRequestEntity`, `TrendReportView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `TrendRequestEntity` | `EventSourcedEntity` | Per-request lifecycle: submitted → data-ready → synthesizing → report-ready → evaluated. Source of truth. | `TrendEndpoint`, `TrendDataFetcher`, `TrendRequestWorkflow` | `TrendReportView` |
| `TrendDataFetcher` | `Consumer` | Subscribes to `TrendQuerySubmitted` events; assembles the matching seeded `RawTrendPayload`; calls `TrendRequestEntity.attachTrendData`. | `TrendRequestEntity` events | `TrendRequestEntity` |
| `TrendRequestWorkflow` | `Workflow` | One workflow per requestId. Steps: `awaitDataStep` → `synthesizeStep` → `evalStep`. | started by `TrendDataFetcher` once data-ready event lands | `TrendsExplorerAgent`, `TrendRequestEntity` |
| `TrendsExplorerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the synthesis task parameters and the raw trend payload as a task attachment; returns `TrendReport`. | invoked by `TrendRequestWorkflow` | returns report |
| `ReportQualityGuardrail` | supporting class | `before-agent-response` hook on `TrendsExplorerAgent`. Validates report structure before it leaves the agent loop. | registered on agent | — |
| `ReportEvaluationScorer` | supporting class | Deterministic scorer for `evalStep`. No LLM call. | invoked by workflow | — |
| `TrendReportView` | `View` | Read model: one row per trend request for the UI. | `TrendRequestEntity` events | `TrendEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TrendQuery(
    String requestId,
    String region,         // BCP-47 region code, e.g. "US", "DE", "JP"
    TimeWindow timeWindow, // enum
    String category,       // e.g. "Technology", "All"
    String submittedBy,
    Instant submittedAt
) {}
enum TimeWindow { PAST_DAY, PAST_WEEK, PAST_MONTH, PAST_90_DAYS }

record RawTrendPayload(
    String region,
    TimeWindow timeWindow,
    String category,
    List<RawTopic> topics,  // raw data rows from the seed dataset
    Instant fetchedAt
) {}

record RawTopic(
    String topicName,
    int searchVolumeIndex,   // 0–100 relative scale
    boolean breakout,        // true if growth > 5000% in window
    List<String> relatedQueries
) {}

record TrendingTopic(
    int rank,
    String topicName,
    int searchVolumeIndex,
    boolean breakout,
    String rationale   // agent's one-sentence explanation of why this topic is trending
) {}

record RelatedCluster(
    String anchorTerm,
    List<String> relatedQueries  // top 3
) {}

record TrendReport(
    String region,
    TimeWindow timeWindow,
    String category,
    List<TrendingTopic> topics,       // ranked 1..N
    String breakoutSummary,           // agent's paragraph on breakout signals
    List<RelatedCluster> clusters,
    Instant reportedAt
) {}

record EvalResult(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record TrendRequest(
    String requestId,
    Optional<TrendQuery> query,
    Optional<RawTrendPayload> rawPayload,
    Optional<TrendReport> report,
    Optional<EvalResult> eval,
    TrendRequestStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum TrendRequestStatus {
    SUBMITTED, DATA_READY, SYNTHESIZING, REPORT_READY, EVALUATED, FAILED
}
```

Events on `TrendRequestEntity`: `TrendQuerySubmitted`, `TrendDataReady`, `SynthesisStarted`, `ReportRecorded`, `EvaluationScored`, `TrendRequestFailed`.

Every nullable lifecycle field on `TrendRequest` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/trends` — body `{ region, timeWindow, category, submittedBy }` → `{ requestId }`.
- `GET /api/trends` — list all requests, newest-first.
- `GET /api/trends/{id}` — one request.
- `GET /api/trends/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Google Trends Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted requests (status pill + region/window label + age) and a right pane with the selected request's detail — query parameters, raw payload summary, report ranked-topic list, breakout signal paragraph, related-query cluster table, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

This baseline has **zero controls** — `eval-matrix.yaml` contains an empty `controls` array and an empty `simplified_view`. The on-decision evaluator (`ReportEvaluationScorer`) runs in `evalStep` as a quality signal, not as an enforcement mechanism; it is a component in PLAN.md but not a registered control. See `eval-matrix.yaml` and `risk-survey.yaml`. A deployer adding PII handling (e.g., user-supplied custom seeds that contain names) should add an S-prefixed sanitizer control.

## 9. Agent prompts

- `TrendsExplorerAgent` → `prompts/trends-explorer.md`. The single synthesis LLM. System prompt instructs it to read the attached raw trend payload, rank the topics, identify breakout signals, group related queries into clusters, and return one `TrendReport`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits US / Past week / Technology; within 30 s the report appears with all ranked topics, a breakout summary, and an eval score chip.
2. **J2** — The agent's first response on a request is intentionally malformed (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a well-formed report; the UI never displays the malformed response.
3. **J3** — A report whose topics all carry empty `rationale` strings receives an eval score of 1 with a clear rationale; the UI flags the card.
4. **J4** — A request for APAC / Past month / Finance produces a report whose `clusters` list has at least one `RelatedCluster`; every `anchorTerm` matches a `topicName` from `topics`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named google-trends-agent demonstrating the single-agent × research-intel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-trends-explorer. Java package io.akka.samples.googletrendsagent.
Akka 3.6.0. HTTP port 9674.

Components to wire (exactly):

- 1 AutonomousAgent TrendsExplorerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/trends-explorer.md>) and
  .capability(TaskAcceptance.of(SYNTHESIZE_TRENDS).maxIterationsPerTask(3)). The task receives
  the query parameters (region, timeWindow, category) as its instruction text and the raw trend
  payload JSON as a task ATTACHMENT named "trend-data.json"
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: TrendReport{region, timeWindow, category, topics: List<TrendingTopic>,
  breakoutSummary: String, clusters: List<RelatedCluster>, reportedAt: Instant}. The agent is
  configured with a before-agent-response guardrail (ReportQualityGuardrail) registered via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries within
  its 3-iteration budget.

- 1 Workflow TrendRequestWorkflow per requestId with three steps:
  * awaitDataStep — polls TrendRequestEntity.getTrendRequest every 1s; on
    request.rawPayload().isPresent() advances to synthesizeStep.
    WorkflowSettings.stepTimeout 15s (fetcher is in-process and fast).
  * synthesizeStep — emits SynthesisStarted, then calls componentClient.forAutonomousAgent(
    TrendsExplorerAgent.class, "explorer-" + requestId).runSingleTask(
      TaskDef.instructions(formatQueryParams(query))
        .attachment("trend-data.json", serializePayload(rawPayload).getBytes())
    ) — returns a taskId, then forTask(taskId).result(SYNTHESIZE_TRENDS) to fetch the report.
    On success calls TrendRequestEntity.recordReport(report).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(TrendRequestWorkflow::error).
  * evalStep — runs a deterministic rule-based ReportEvaluationScorer (NOT an LLM call) over
    the recorded report: checks that every TrendingTopic has non-empty rationale, that
    breakoutSummary is non-empty, that every RelatedCluster.anchorTerm matches a topicName in
    the report's topics list, and that topics are strictly ranked (rank 1..N with no gaps).
    Emits EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity TrendRequestEntity (one per requestId). State TrendRequest{requestId:
  String, query: Optional<TrendQuery>, rawPayload: Optional<RawTrendPayload>,
  report: Optional<TrendReport>, eval: Optional<EvalResult>, status: TrendRequestStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. TrendRequestStatus enum: SUBMITTED,
  DATA_READY, SYNTHESIZING, REPORT_READY, EVALUATED, FAILED. Events: TrendQuerySubmitted{query},
  TrendDataReady{rawPayload}, SynthesisStarted{}, ReportRecorded{report},
  EvaluationScored{eval}, TrendRequestFailed{reason}. Commands: submit, attachTrendData,
  markSynthesizing, recordReport, recordEvaluation, fail, getTrendRequest.
  emptyState() returns TrendRequest.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer TrendDataFetcher subscribed to TrendRequestEntity events; on TrendQuerySubmitted
  looks up the matching seeded dataset from src/main/resources/sample-events/trend-data.jsonl
  (keyed by region + timeWindow + category), builds a RawTrendPayload, then calls
  TrendRequestEntity.attachTrendData(rawPayload). After attachTrendData lands, the same Consumer
  starts a TrendRequestWorkflow with id = "trend-" + requestId.

- 1 View TrendReportView with row type TrendReportRow (mirrors TrendRequest minus the full
  RawTrendPayload body — the view stores payload metadata only: region, timeWindow, category,
  topicCount). Table updater consumes TrendRequestEntity events. ONE query getAllRequests:
  SELECT * AS requests FROM trend_report_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * TrendEndpoint at /api with POST /trends (body {region, timeWindow, category, submittedBy};
    mints requestId; calls TrendRequestEntity.submit; returns {requestId}), GET /trends
    (list from getAllRequests, sorted newest-first), GET /trends/{id} (one row), GET
    /trends/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- TrendTasks.java declaring one Task<R> constant: SYNTHESIZE_TRENDS = Task.name("Synthesize
  trends").description("Read the attached trend-data payload and produce a TrendReport with
  ranked topics, breakout signals, and related query clusters")
  .resultConformsTo(TrendReport.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records TrendQuery, TimeWindow, RawTrendPayload, RawTopic, TrendingTopic,
  RelatedCluster, TrendReport, EvalResult, TrendRequest, TrendRequestStatus.

- ReportQualityGuardrail.java implementing the before-agent-response hook. Reads the candidate
  TrendReport from the LLM response, checks (1) topics list is non-empty, (2) every
  TrendingTopic has non-empty topicName and non-empty rationale, (3) every rank is in 1..N
  with no duplicate ranks, (4) breakoutSummary is non-empty. On any failure returns
  Guardrail.reject(<structured-error>) to force the agent loop to retry.

- ReportEvaluationScorer.java — pure deterministic logic (no LLM). Inputs: TrendReport.
  Outputs: EvalResult. Scoring: +1 if every TrendingTopic.rationale is non-empty (max 2
  points if all are ≥ 10 words), +1 if breakoutSummary is ≥ 20 words, +1 if clusters list
  is non-empty, +1 if every RelatedCluster.anchorTerm matches a topicName in topics. Score
  capped at 5. Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9674 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The TrendsExplorerAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/trend-data.jsonl with seeded datasets for at minimum:
  US/PAST_WEEK/Technology, EU/PAST_MONTH/Finance, APAC/PAST_90_DAYS/Health,
  LATAM/PAST_DAY/Entertainment, Global/PAST_WEEK/All. Each dataset has 8–12 RawTopic rows
  with realistic topic names, searchVolumeIndex values, breakout flags, and 3–5 related
  queries per topic. At least 2 topics per dataset carry breakout=true.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 0 controls and an empty simplified_view list.

- risk-survey.yaml at the project root pre-filled for research-intel domain, with deployer
  fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/trends-explorer.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Google Trends Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of request cards; right = selected-request detail with query parameters, raw
  payload summary, ranked topic list, breakout signal paragraph, related-cluster table, and
  eval-score chip). Browser title exactly:
  <title>Akka Sample: Google Trends Agent</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(requestId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    synthesize-trends.json — 8 TrendReport entries covering realistic topic distributions.
      Each entry has a non-empty breakoutSummary, a topics list of 6–10 TrendingTopic rows
      (each with non-empty topicName, valid rank, searchVolumeIndex 10–100, and non-empty
      rationale of ≥ 10 words), and a clusters list whose anchorTerms match topic names.
      Plus 2 deliberately MALFORMED entries (one with a topic whose rationale is empty; one
      with duplicate rank values) — the guardrail blocks both, exercising the retry path.
      The mock should select a malformed entry on the FIRST iteration of every 3rd request
      (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(requestId) helper makes per-request selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. TrendsExplorerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion TrendTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (synthesizeStep
  60s, awaitDataStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the TrendRequest row record is Optional<T>.
- Lesson 7: TrendTasks.java with SYNTHESIZE_TRENDS = Task.name(...).description(...)
  .resultConformsTo(TrendReport.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9674 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (TrendsExplorerAgent).
  The on-decision evaluator is rule-based (ReportEvaluationScorer.java) and does NOT make
  an LLM call — keeping the pattern's "one agent" promise honest.
- The trend payload is passed as a Task ATTACHMENT named "trend-data.json", never inlined
  into the agent's instructions. Verify the generated synthesizeStep uses
  TaskDef.attachment(...) and not string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
