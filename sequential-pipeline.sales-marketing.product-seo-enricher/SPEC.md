# SPEC — product-seo-enricher

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Product SEO Enricher.
**One-line pitch:** A user submits a product identifier; one `SeoAgent` walks it through three task phases — **FETCH** top search-engine result pages, **ANALYZE** them for keyword signals and competitor positioning, **ENRICH** the product record with a ranked keyword list, competitor summary, and a recommended meta description — with each phase gated on the prior phase's recorded output and browser-tool access rejected when called out of order.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a sales-marketing SEO enrichment domain. One `SeoAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the FETCH task's typed output (`SerpResult`) becomes the ANALYZE task's instruction context; the ANALYZE task's typed output (`SerpAnalysis`) becomes the ENRICH task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. It looks at the call's declared phase (`FETCH` / `ANALYZE` / `ENRICH`) and the current `EnrichmentEntity` status. An ENRICH-phase tool called while the entity has not yet recorded `SerpAnalyzed` is rejected before the tool body runs. The rejection returns a structured error to the agent so the task loop can correct course inside its iteration budget. This is the mechanism that enforces the browser-tool access policy: the FETCH phase holds the only tools that touch raw search-result content; downstream phases receive only structured, already-processed data.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce both the dependency contract and per-phase browser-tool isolation.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **product name** (or picks one of three seeded products — `UltraFit Running Shoe X9`, `MindClear Meditation App Pro`, `GardenPro Raised Bed Kit`).
2. The user clicks **Run enrichment**. The UI POSTs to `/api/enrichments` and receives an `enrichmentId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `FETCHING` — the workflow has started `fetchStep` and the agent has been handed the FETCH task.
4. Within ~10–20 s the card reaches `ANALYZING` — the typed `SerpResult` is visible in the card detail (a small table of SERP entries with title, url, and position). The agent's FETCH task returned; the workflow recorded `SerpFetched` and ran the ANALYZE task.
5. Within ~10–20 s more the card reaches `ENRICHING`. The `SerpAnalysis` is visible (keyword candidates list + competitor signals).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `ProductEnrichment` — recommended keywords ranked by relevance score, competitor positioning summary, and the recommended meta description — plus a quality score chip (1–5) and a one-line rationale.
7. The user can submit another product; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `EnrichmentEndpoint` | `HttpEndpoint` | `/api/enrichments/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `EnrichmentEntity`, `EnrichmentView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `EnrichmentEntity` | `EventSourcedEntity` | Per-product lifecycle: created → fetching → fetched → analyzing → analyzed → enriching → enriched → evaluated. Source of truth. | `EnrichmentEndpoint`, `EnrichmentPipelineWorkflow` | `EnrichmentView` |
| `EnrichmentPipelineWorkflow` | `Workflow` | One workflow per enrichment. Steps: `fetchStep` → `analyzeStep` → `enrichStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `EnrichmentEndpoint` after `CREATED` | `SeoAgent`, `EnrichmentEntity` |
| `SeoAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `SeoTasks.java`: `FETCH_SERP` → `SerpResult`, `ANALYZE_SERP` → `SerpAnalysis`, `WRITE_ENRICHMENT` → `ProductEnrichment`. Each task is registered with the phase-appropriate function tools. | invoked by `EnrichmentPipelineWorkflow` | returns typed results |
| `FetchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `fetchSerpPage(product)` and `fetchResultSnippet(url)`. Reads from `src/main/resources/sample-data/serp/*.json` for deterministic offline output. | called from FETCH task | returns `List<SerpEntry>` |
| `AnalyzeTools` | function-tools class | Implements `extractKeywords(serpEntries)` and `identifyCompetitors(serpEntries)`. Pure in-memory transformations. | called from ANALYZE task | returns `List<KeywordCandidate>` / `List<CompetitorSignal>` |
| `EnrichTools` | function-tools class | Implements `rankKeywords(keywords, analysis)` and `writeMetaDescription(product, analysis)`. | called from ENRICH task | returns `List<RankedKeyword>` / `String` |
| `BrowserGuardrail` | `before-tool-call` guardrail (registered on `SeoAgent`) | Reads the in-flight task's declared phase and the current `EnrichmentEntity` status. Rejects any tool call whose phase precondition has not been satisfied. | every tool call on every task | accept / structured-reject |
| `QualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `ProductEnrichment`, `SerpAnalysis`, `SerpResult`. Output: `QualityResult{score, rationale}`. | called from `evalStep` | returns score |
| `EnrichmentView` | `View` | Read model: one row per enrichment for the UI. | `EnrichmentEntity` events | `EnrichmentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SerpEntry(
    String title,
    String url,
    int position,
    String snippet,
    Instant fetchedAt
) {}

record SerpResult(List<SerpEntry> entries, Instant fetchedAt) {}

record KeywordCandidate(String keyword, int frequency, String sourceUrl) {}

record CompetitorSignal(String domain, String title, int position, String angle) {}

record SerpAnalysis(
    List<KeywordCandidate> keywords,
    List<CompetitorSignal> competitors,
    Instant analyzedAt
) {}

record RankedKeyword(String keyword, double relevanceScore, String rationale) {}

record ProductEnrichment(
    String productName,
    List<RankedKeyword> rankedKeywords,
    String competitorSummary,
    String recommendedMetaDescription,
    Instant writtenAt
) {}

record QualityResult(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record EnrichmentRecord(
    String enrichmentId,
    Optional<String> productName,
    Optional<SerpResult> serpResult,
    Optional<SerpAnalysis> serpAnalysis,
    Optional<ProductEnrichment> enrichment,
    Optional<QualityResult> quality,
    EnrichmentStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EnrichmentStatus {
    CREATED, FETCHING, FETCHED, ANALYZING, ANALYZED,
    ENRICHING, ENRICHED, EVALUATED, FAILED
}
```

Events on `EnrichmentEntity`: `EnrichmentCreated`, `FetchStarted`, `SerpFetched`, `AnalyzeStarted`, `SerpAnalyzed`, `EnrichStarted`, `EnrichmentWritten`, `QualityScored`, `GuardrailRejected`, `EnrichmentFailed`.

Every nullable lifecycle field on the `EnrichmentRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/enrichments` — body `{ productName }` → `{ enrichmentId }`.
- `GET /api/enrichments` — list all enrichments, newest-first.
- `GET /api/enrichments/{id}` — one enrichment.
- `GET /api/enrichments/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Product SEO Enricher</title>`.

The App UI tab is a two-column layout: a left rail with the live list of enrichments (status pill + product name + age) and a right pane with the selected enrichment's detail — SERP table, keyword candidates, competitor signals, ranked keywords, meta description, quality score chip, and a guardrail-rejection log strip if any phase-gate rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (browser-tool phase-gate)**: `BrowserGuardrail` is registered on `SeoAgent` and runs before every tool call. It reads the in-flight `Task`'s declared phase (encoded as a constant on each function-tool class — `Phase.FETCH`, `Phase.ANALYZE`, `Phase.ENRICH`) and the current `EnrichmentEntity.status` for the enrichment the task is bound to. The accept rule is precise: `FETCH` tools require `status ∈ {CREATED, FETCHING}`; `ANALYZE` tools require `status ∈ {FETCHED, ANALYZING}` AND `serpResult.isPresent()`; `ENRICH` tools require `status ∈ {ANALYZED, ENRICHING}` AND `serpAnalysis.isPresent()`. On reject, the guardrail returns a structured `phase-violation` error to the agent loop and the workflow records a `GuardrailRejected{phase, tool, reason}` event for visibility. The agent loop retries within its 4-iteration budget. This mechanism prevents downstream phases from directly accessing search-result page content — only `FetchTools` can do that; `AnalyzeTools` and `EnrichTools` receive only the already-structured outputs.

## 9. Agent prompts

- `SeoAgent` → `prompts/seo-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded product `UltraFit Running Shoe X9`; within 60 s the enrichment reaches `EVALUATED` with non-empty SERP entries, ≥ 3 keyword candidates, ≥ 1 ranked keyword, and a quality score chip on the card.
2. **J2** — The agent's first iteration on an enrichment calls an ENRICH-phase tool (`rankKeywords`) before `SerpAnalyzed` has been recorded (mock LLM path). `BrowserGuardrail` rejects the call; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the enrichment eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — An enrichment whose mock-LLM trajectory produces a ranked keyword with no backing `KeywordCandidate` in the recorded `SerpAnalysis` is scored 1; the UI flags the card.
4. **J4** — Each task's tool calls are visible in the per-enrichment trace; the FETCH task's log shows only FetchTools calls, the ANALYZE task's log shows only AnalyzeTools calls, the ENRICH task's log shows only EnrichTools calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named product-seo-enricher demonstrating the sequential-pipeline x
sales-marketing cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-sales-marketing-product-seo-enricher.
Java package io.akka.samples.brandsearchoptimization. Akka 3.6.0. HTTP port 9105.

Components to wire (exactly):

- 1 AutonomousAgent SeoAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/seo-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  FETCH, ANALYZE, and ENRICH tool sets are ALL registered on the agent; phase gating is the
  job of BrowserGuardrail, NOT of conditional .tools(...) wiring. The before-tool-call
  guardrail (BrowserGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow EnrichmentPipelineWorkflow per enrichmentId with four steps:
  * fetchStep — emits FetchStarted on the entity, then calls componentClient
    .forAutonomousAgent(SeoAgent.class, "agent-" + enrichmentId).runSingleTask(
      TaskDef.instructions("Product: " + productName + "\nPhase: FETCH\nUse fetchSerpPage
      and fetchResultSnippet to collect 5-10 SERP entries for this product.")
        .metadata("enrichmentId", enrichmentId)
        .metadata("phase", "FETCH")
        .taskType(SeoTasks.FETCH_SERP)
    ). Reads forTask(taskId).result(FETCH_SERP) to get SerpResult. Writes
    EnrichmentEntity.recordSerpResult(serpResult). WorkflowSettings.stepTimeout 60s.
  * analyzeStep — emits AnalyzeStarted, then runSingleTask with TaskDef.instructions
    (formatAnalyzeContext(serpResult, productName)) and metadata.phase = "ANALYZE", taskType
    ANALYZE_SERP. Writes EnrichmentEntity.recordSerpAnalysis(serpAnalysis). stepTimeout 60s.
  * enrichStep — emits EnrichStarted, then runSingleTask with TaskDef.instructions
    (formatEnrichContext(serpAnalysis, serpResult, productName)) and metadata.phase = "ENRICH",
    taskType WRITE_ENRICHMENT. Writes EnrichmentEntity.recordEnrichment(enrichment).
    stepTimeout 60s.
  * evalStep — runs the deterministic QualityScorer over (enrichment, serpAnalysis, serpResult)
    and writes EnrichmentEntity.recordQuality(quality). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(EnrichmentPipelineWorkflow::error). The error step writes
  EnrichmentFailed and ends.

- 1 EventSourcedEntity EnrichmentEntity (one per enrichmentId). State EnrichmentRecord{
  enrichmentId, productName: Optional<String>, serpResult: Optional<SerpResult>,
  serpAnalysis: Optional<SerpAnalysis>, enrichment: Optional<ProductEnrichment>,
  quality: Optional<QualityResult>, status: EnrichmentStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. EnrichmentStatus enum: CREATED, FETCHING, FETCHED,
  ANALYZING, ANALYZED, ENRICHING, ENRICHED, EVALUATED, FAILED. Events:
  EnrichmentCreated{productName}, FetchStarted, SerpFetched{serpResult}, AnalyzeStarted,
  SerpAnalyzed{serpAnalysis}, EnrichStarted, EnrichmentWritten{enrichment},
  QualityScored{quality}, GuardrailRejected{phase, tool, reason}, EnrichmentFailed{reason}.
  Commands: create, startFetch, recordSerpResult, startAnalyze, recordSerpAnalysis,
  startEnrich, recordEnrichment, recordQuality, recordGuardrailRejection, fail,
  getEnrichment. emptyState() returns EnrichmentRecord.initial("") with all Optional fields
  as Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View EnrichmentView with row type EnrichmentRow that mirrors EnrichmentRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes EnrichmentEntity
  events. ONE query getAllEnrichments: SELECT * AS enrichments FROM enrichment_view. No
  WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * EnrichmentEndpoint at /api with POST /enrichments (body {productName}; mints
    enrichmentId; calls EnrichmentEntity.create(productName); then starts
    EnrichmentPipelineWorkflow with id "pipeline-" + enrichmentId; returns {enrichmentId}),
    GET /enrichments (list from getAllEnrichments, sorted newest-first), GET /enrichments/{id}
    (one row), GET /enrichments/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- SeoTasks.java declaring three Task<R> constants:
    FETCH_SERP = Task.name("Fetch SERP").description("Retrieve top search-engine result
      entries for a product by calling fetchSerpPage and fetchResultSnippet")
      .resultConformsTo(SerpResult.class);
    ANALYZE_SERP = Task.name("Analyze SERP").description("Extract keyword candidates and
      identify competitor signals from the SERP entries")
      .resultConformsTo(SerpAnalysis.class);
    WRITE_ENRICHMENT = Task.name("Write enrichment").description("Rank keyword candidates
      by relevance and compose a recommended meta description")
      .resultConformsTo(ProductEnrichment.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Phase.java — enum {FETCH, ANALYZE, ENRICH}. Each function-tool method is annotated with
  the constant phase, e.g. @FunctionTool(name = "fetchSerpPage", phase = Phase.FETCH)
  (use a custom annotation if the SDK's @FunctionTool does not carry a phase field — the
  guardrail reads it from a parallel registry built at startup if so).

- FetchTools.java — @FunctionTool fetchSerpPage(String product) -> List<SerpEntry> reading
  from src/main/resources/sample-data/serp/*.json keyed by product slug; @FunctionTool
  fetchResultSnippet(String url) -> String reading the matching entry's snippet field.

- AnalyzeTools.java — @FunctionTool extractKeywords(List<SerpEntry>) -> List<KeywordCandidate>
  (one candidate per significant term appearing in ≥ 2 entries, frequency = occurrence count,
  sourceUrl = first entry's url containing that term); @FunctionTool
  identifyCompetitors(List<SerpEntry>) -> List<CompetitorSignal> (one signal per distinct
  domain in entries, position = that domain's best rank, angle extracted from its title).

- EnrichTools.java — @FunctionTool rankKeywords(List<KeywordCandidate>, SerpAnalysis)
  -> List<RankedKeyword> (score on 0.0–1.0 scale: frequency / max_frequency, rationale is
  one clause naming the backing evidence); @FunctionTool writeMetaDescription(String product,
  SerpAnalysis) -> String (≤ 160 chars, incorporates the top-3 ranked keywords).

- BrowserGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's @FunctionTool.phase attribute, looks up the EnrichmentEntity status by enrichmentId
  (carried in the TaskDef metadata), applies the accept matrix from Section 8, and either
  passes or returns Guardrail.reject("phase-violation: <tool> requires <precondition>, saw
  <status>"). On reject ALSO calls EnrichmentEntity.recordGuardrailRejection(phase, tool,
  reason) so the rejection is visible in the UI's rejection-log strip and in the audit log.

- QualityScorer.java — pure deterministic logic (no LLM). Inputs: ProductEnrichment,
  SerpAnalysis, SerpResult. Outputs: QualityResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1: keyword coverage (every
  RankedKeyword.keyword appears in SerpAnalysis.keywords[].keyword), meta-description length
  (recommendedMetaDescription.length() ≤ 160), competitor grounding (competitorSummary
  references at least one CompetitorSignal.domain), and keyword ranking non-empty
  (rankedKeywords.size() > 0). Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9105 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/products.jsonl with 5 seeded product lines covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/serp/*.json — three files keyed by seeded product slug,
  each carrying 6-10 SerpEntry records with deterministic content so FetchTools.fetchSerpPage
  returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (product names
  are catalog-level, not person-level), decisions.authority_level = recommend-only (the
  enrichment is advisory to the content team), oversight.human_in_loop = true (a content
  editor reviews before publishing), operations.agent_count = 1, operations.agent_pattern =
  sequential-pipeline, failure.failure_modes including "phase-violated-browser-tool",
  "missing-keyword-coverage", "hallucinated-competitor", "meta-description-too-long";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/seo-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Product SEO Enricher", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of enrichment cards; right = selected-enrichment detail with product header,
  SERP entries table, keyword candidates, competitor signals, ranked keywords, meta
  description, quality-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Product SEO Enricher</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        typed-correct outputs per Task. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(enrichmentId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    fetch-serp.json — 6 SerpResult entries, each with 5-8 SerpEntry items per seeded
      product. Each entry's tool_calls array contains 2-4 calls: 1 fetchSerpPage(product) +
      1-3 fetchResultSnippet(url) calls. Plus 1 deliberately PHASE-VIOLATING entry whose
      tool_calls array starts with a rankKeywords(...) call (an ENRICH-phase tool called
      during the FETCH phase) — the guardrail rejects it, the mock then falls through to a
      normal fetch sequence. The mock should select the violating entry on the FIRST
      iteration of every 3rd enrichment (modulo seed) so J2 is reproducible.
    analyze-serp.json — 6 SerpAnalysis entries paired one-to-one with the fetch entries,
      each with 4-8 KeywordCandidate items and 2-4 CompetitorSignal items, with tool_calls
      containing extractKeywords + identifyCompetitors in order.
    write-enrichment.json — 6 ProductEnrichment entries paired one-to-one. Each carries
      3-6 RankedKeyword items whose keywords all appear in the paired SerpAnalysis.keywords,
      a competitorSummary naming at least one domain, a recommendedMetaDescription ≤ 160
      chars, tool_calls containing rankKeywords + writeMetaDescription. Plus 1 deliberately
      UNGROUNDED-KEYWORD entry whose first RankedKeyword references a keyword absent from
      the paired SerpAnalysis.keywords — the evalStep scores it 1; J3 verifies this.
- A MockModelProvider.seedFor(enrichmentId) helper makes per-enrichment selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SeoAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SeoTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (fetchStep
  60s, analyzeStep 60s, enrichStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the EnrichmentRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: SeoTasks.java with FETCH_SERP, ANALYZE_SERP, WRITE_ENRICHMENT constants is
  mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9105 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (SeoAgent). The
  on-decision eval is rule-based (QualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the before-tool-call guardrail (BrowserGuardrail) is the runtime mechanism that enforces
  the phase order.
- Task dependency is carried by typed task results: fetchStep writes SerpResult onto the
  entity, analyzeStep reads it and builds the ANALYZE task's instruction context from it,
  enrichStep reads both. The agent itself is stateless across phases.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
