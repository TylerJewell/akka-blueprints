# SPEC — competitor-research-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Competitor Research Pipeline.
**One-line pitch:** A user submits a competitor name; one `ResearchAgent` walks it through three task phases — **SEARCH** web results via Exa.ai, **SUMMARIZE** findings into structured profile fields, **PUBLISH** a typed `CompetitorProfile` into Notion — with each phase gated on the prior phase's recorded output and the Notion write validated against schema and scope constraints before it executes.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a research-intelligence domain. One `ResearchAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the SEARCH task's typed output becomes the SUMMARIZE task's instruction context; the SUMMARIZE task's typed output becomes the PUBLISH task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- A **`before-tool-call` guardrail** sits between the agent and every tool call. At the phase boundary it checks preconditions exactly as in the general pipeline template: a PUBLISH-phase tool called before `SummaryProduced` has been recorded is rejected, and the rejection returns a structured error to the agent loop so it can correct course within its iteration budget. For PUBLISH-phase tools specifically, the guardrail also runs a **schema and scope validation**: the Notion write payload must include every required field declared in `NotionSchema.REQUIRED_FIELDS`, every field value must match the declared Notion property type, and the target database ID must equal `NotionSchema.ALLOWED_DATABASE_ID`. A payload that fails any of these three sub-checks is rejected with a structured `schema-violation` or `scope-violation` error before the Notion API is called. This validation matters because writing a malformed record to Notion corrupts the competitor database that downstream analysts depend on, and there is no built-in rollback in Notion's API.

The blueprint shows that for a pipeline that terminates in an external write, the `before-tool-call` hook is the right place to enforce schema and scope rules — catching the violation at the agent–tool boundary is cheaper and more auditable than catching it after the write has partially executed.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **competitor name** into the input (or picks one of three seeded competitors — `Acme Analytics`, `Beacon DataOps`, `Crestline AI`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/competitors` and receives a `competitorId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `SEARCHING` — the workflow has started `searchStep` and the agent has been handed the SEARCH task.
4. Within ~15–30 s the card reaches `SUMMARIZING` — the typed `SearchResultSet` is visible in the card detail (a small table of results with title, url, and excerpt). The agent's SEARCH task returned; the workflow recorded `ResultsFetched` and ran the SUMMARIZE task.
5. Within ~15–30 s more the card reaches `PUBLISHING`. The `ProfileSummary` is visible (field list: pricing model, primary use case, notable integrations, known differentiators, data residency stance).
6. Within ~15–30 s more the card reaches `EVALUATED`. The right pane now shows the full typed `CompetitorProfile` — name, one-line description, per-field profile entries, and a Notion page URL — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another competitor; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CompetitorEndpoint` | `HttpEndpoint` | `/api/competitors/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `CompetitorEntity`, `CompetitorView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CompetitorEntity` | `EventSourcedEntity` | Per-competitor lifecycle: created → searching → searched → summarizing → summarized → publishing → published → evaluated. Source of truth. | `CompetitorEndpoint`, `CompetitorPipelineWorkflow` | `CompetitorView` |
| `CompetitorPipelineWorkflow` | `Workflow` | One workflow per competitorId. Steps: `searchStep` → `summarizeStep` → `publishStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `CompetitorEndpoint` after `CREATED` | `ResearchAgent`, `CompetitorEntity` |
| `ResearchAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `ResearchTasks.java`: `SEARCH_COMPETITOR` → `SearchResultSet`, `SUMMARIZE_FINDINGS` → `ProfileSummary`, `PUBLISH_PROFILE` → `CompetitorProfile`. Each task is registered with the phase-appropriate function tools. | invoked by `CompetitorPipelineWorkflow` | returns typed results |
| `SearchTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchCompetitor(name)` and `fetchPage(url)`. Reads from `src/main/resources/sample-data/competitors/*.json` for deterministic offline output; calls Exa.ai in live mode. | called from SEARCH task | returns `List<SearchResult>` |
| `SummarizeTools` | function-tools class | Implements `extractFields(results)` and `classifyDomain(results)`. Pure in-memory transformations that distil raw web excerpts into typed profile fields. | called from SUMMARIZE task | returns `List<ProfileField>` / `String` |
| `PublishTools` | function-tools class | Implements `buildNotionPage(summary)` and `writeToNotion(page)`. `writeToNotion` is the only tool that calls an external service (Notion API); it is gated by `NotionWriteGuardrail`. | called from PUBLISH task | returns `NotionPageRef` |
| `NotionWriteGuardrail` | `before-tool-call` guardrail (registered on `ResearchAgent`) | Phase-precondition check (same as general template) PLUS schema-and-scope validation on every `writeToNotion` call: required fields present, field types match `NotionSchema` constants, target database ID equals `NotionSchema.ALLOWED_DATABASE_ID`. | every tool call on every task | accept / structured-reject |
| `PublishQualityScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `CompetitorProfile`, `ProfileSummary`, `SearchResultSet`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `CompetitorView` | `View` | Read model: one row per competitor for the UI. | `CompetitorEntity` events | `CompetitorEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SearchResult(String title, String url, String excerpt, Instant fetchedAt) {}

record SearchResultSet(List<SearchResult> results, Instant searchedAt) {}

record ProfileField(String fieldName, String value, String supportingUrl) {}

record ProfileSummary(
    List<ProfileField> fields,
    String domainClassification,
    Instant summarizedAt
) {}

record NotionPageRef(String pageId, String pageUrl, Instant publishedAt) {}

record CompetitorProfile(
    String competitorId,
    String name,
    String oneLiner,
    List<ProfileField> fields,
    NotionPageRef notionRef,
    Instant profiledAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record CompetitorRecord(
    String competitorId,
    Optional<String> name,
    Optional<SearchResultSet> searchResults,
    Optional<ProfileSummary> summary,
    Optional<CompetitorProfile> profile,
    Optional<EvalResult> eval,
    CompetitorStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum CompetitorStatus {
    CREATED, SEARCHING, SEARCHED, SUMMARIZING, SUMMARIZED,
    PUBLISHING, PUBLISHED, EVALUATED, FAILED
}
```

Events on `CompetitorEntity`: `CompetitorCreated`, `SearchStarted`, `ResultsFetched`, `SummarizeStarted`, `SummaryProduced`, `PublishStarted`, `ProfilePublished`, `PublishEvaluated`, `GuardrailRejected`, `CompetitorFailed`.

Every nullable lifecycle field on the `CompetitorRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/competitors` — body `{ name }` → `{ competitorId }`.
- `GET /api/competitors` — list all competitors, newest-first.
- `GET /api/competitors/{id}` — one competitor record.
- `GET /api/competitors/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Competitor Research Pipeline</title>`.

The App UI tab is a two-column layout: a left rail with the live list of competitors (status pill + name + age) and a right pane with the selected competitor's detail — name, search results table, profile summary fields, full profile, eval score chip, and a guardrail-rejection log strip if any phase-gate or schema rejections occurred.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-tool-call` guardrail (phase-gate + Notion schema/scope check)**: `NotionWriteGuardrail` is registered on `ResearchAgent` and runs before every tool call. For phase gating, the accept matrix mirrors the general template: `SEARCH` tools require `status ∈ {CREATED, SEARCHING}`; `SUMMARIZE` tools require `status ∈ {SEARCHED, SUMMARIZING}` AND `searchResults.isPresent()`; `PUBLISH` tools require `status ∈ {SUMMARIZED, PUBLISHING}` AND `summary.isPresent()`. On phase-gate reject, the guardrail returns a structured `phase-violation` error and records `GuardrailRejected{phase, tool, reason}` on the entity. For `writeToNotion` calls (regardless of phase gating), the guardrail additionally checks: (a) all `NotionSchema.REQUIRED_FIELDS` are present in the payload, (b) each field value's type matches the corresponding Notion property type in `NotionSchema.FIELD_TYPES`, and (c) the target `databaseId` equals `NotionSchema.ALLOWED_DATABASE_ID`. A failure on any sub-check produces a `schema-violation` or `scope-violation` error and records `GuardrailRejected` for audit. The agent loop retries within its 4-iteration budget.

## 9. Agent prompts

- `ResearchAgent` → `prompts/research-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded competitor `Acme Analytics`; within 90 s the pipeline reaches `EVALUATED` with non-empty search results, ≥ 3 profile fields, a valid Notion page URL, and an eval score chip on the card.
2. **J2** — The agent's first iteration on a competitor calls `writeToNotion` before `SummaryProduced` has been recorded (mock LLM path). `NotionWriteGuardrail` rejects the call with a `phase-violation` error; a `GuardrailRejected` event lands on the entity; the agent retries in-phase; the competitor eventually completes correctly. The UI's rejection-log strip shows the one rejected call.
3. **J3** — A `writeToNotion` call whose payload omits the `pricing_model` required field is caught by `NotionWriteGuardrail` with a `schema-violation` error; the mock records the rejection; the agent re-forms the payload with all required fields and the corrected write proceeds.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-competitor trace (logged at `INFO`); the SEARCH task's log shows only SEARCH-tool calls, the SUMMARIZE task's log shows only SUMMARIZE-tool calls, the PUBLISH task's log shows only PUBLISH-tool calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named competitor-research-pipeline demonstrating the sequential-pipeline x
research-intel cell. Requires Exa.ai API key (EXA_API_KEY) and Notion credentials
(NOTION_API_TOKEN, NOTION_DATABASE_ID) for live mode; mock mode is fully self-contained.
Maven group io.akka.samples. Maven artifact
sequential-pipeline-research-intel-competitor-research-pipeline.
Java package io.akka.samples.automatecompetitorresearchwithexaainotionandaiagents.
Akka 3.6.0. HTTP port 9901.

Components to wire (exactly):

- 1 AutonomousAgent ResearchAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/research-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  SEARCH, SUMMARIZE, and PUBLISH tool sets are ALL registered on the agent; phase gating and
  schema/scope validation are the job of NotionWriteGuardrail, NOT of conditional .tools(...)
  wiring. The before-tool-call guardrail (NotionWriteGuardrail) is registered on the agent via
  the agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  within its 4-iteration budget.

- 1 Workflow CompetitorPipelineWorkflow per competitorId with four steps:
  * searchStep — emits SearchStarted on the entity, then calls componentClient
    .forAutonomousAgent(ResearchAgent.class, "agent-" + competitorId).runSingleTask(
      TaskDef.instructions("Competitor: " + name + "\nPhase: SEARCH\nUse searchCompetitor and
      fetchPage to collect 4-8 results about this competitor.")
        .metadata("competitorId", competitorId)
        .metadata("phase", "SEARCH")
        .taskType(ResearchTasks.SEARCH_COMPETITOR)
    ). Reads forTask(taskId).result(SEARCH_COMPETITOR) to get SearchResultSet. Writes
    CompetitorEntity.recordResults(searchResultSet). WorkflowSettings.stepTimeout 90s.
  * summarizeStep — emits SummarizeStarted, then runSingleTask with TaskDef.instructions
    (formatSummarizeContext(searchResultSet, name)) and metadata.phase = "SUMMARIZE", taskType
    SUMMARIZE_FINDINGS. Writes CompetitorEntity.recordSummary(summary). stepTimeout 90s.
  * publishStep — emits PublishStarted, then runSingleTask with TaskDef.instructions
    (formatPublishContext(summary, searchResultSet, name)) and metadata.phase = "PUBLISH", taskType
    PUBLISH_PROFILE. Writes CompetitorEntity.recordProfile(profile). stepTimeout 90s.
  * evalStep — runs the deterministic PublishQualityScorer over (profile, summary,
    searchResults) and writes CompetitorEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(CompetitorPipelineWorkflow::error). The error step writes
  CompetitorFailed and ends.

- 1 EventSourcedEntity CompetitorEntity (one per competitorId). State CompetitorRecord{
  competitorId, name: Optional<String>, searchResults: Optional<SearchResultSet>,
  summary: Optional<ProfileSummary>, profile: Optional<CompetitorProfile>,
  eval: Optional<EvalResult>, status: CompetitorStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. CompetitorStatus enum: CREATED, SEARCHING, SEARCHED,
  SUMMARIZING, SUMMARIZED, PUBLISHING, PUBLISHED, EVALUATED, FAILED. Events:
  CompetitorCreated{name}, SearchStarted, ResultsFetched{searchResults}, SummarizeStarted,
  SummaryProduced{summary}, PublishStarted, ProfilePublished{profile},
  PublishEvaluated{eval}, GuardrailRejected{phase, tool, reason}, CompetitorFailed{reason}.
  Commands: create, startSearch, recordResults, startSummarize, recordSummary, startPublish,
  recordProfile, recordEvaluation, recordGuardrailRejection, fail, getCompetitor. emptyState()
  returns CompetitorRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View CompetitorView with row type CompetitorRow that mirrors CompetitorRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes CompetitorEntity events. ONE
  query getAllCompetitors: SELECT * AS competitors FROM competitor_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * CompetitorEndpoint at /api with POST /competitors (body {name}; mints competitorId; calls
    CompetitorEntity.create(name); then starts CompetitorPipelineWorkflow with id
    "pipeline-" + competitorId; returns {competitorId}), GET /competitors (list from
    getAllCompetitors, sorted newest-first), GET /competitors/{id} (one row), GET
    /competitors/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- ResearchTasks.java declaring three Task<R> constants:
    SEARCH_COMPETITOR = Task.name("Search competitor").description("Gather raw web results
      about a competitor by calling searchCompetitor and fetchPage").resultConformsTo(
      SearchResultSet.class);
    SUMMARIZE_FINDINGS = Task.name("Summarize findings").description("Extract structured
      profile fields and classify the competitor's domain from raw results").resultConformsTo(
      ProfileSummary.class);
    PUBLISH_PROFILE = Task.name("Publish profile").description("Build a Notion page from the
      profile summary and write it to the Notion competitor database").resultConformsTo(
      CompetitorProfile.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- ResearchPhase.java — enum {SEARCH, SUMMARIZE, PUBLISH}. Each function-tool method is
  annotated with the constant phase (use a custom annotation or a parallel registry if the
  SDK's @FunctionTool does not carry a phase field — the guardrail reads the registry at
  startup).

- SearchTools.java — @FunctionTool searchCompetitor(String name) -> List<SearchResult> reading
  from src/main/resources/sample-data/competitors/*.json keyed by name in mock mode; calling
  Exa.ai search API in live mode using EXA_API_KEY from env.
  @FunctionTool fetchPage(String url) -> String reading from the matching result entry's
  excerpt in mock mode; fetching via Exa.ai content API in live mode.

- SummarizeTools.java — @FunctionTool extractFields(List<SearchResult>) -> List<ProfileField>
  (one ProfileField per detected profile dimension: pricing_model, primary_use_case,
  notable_integrations, known_differentiators, data_residency_stance; fieldName is the
  dimension slug, value is the extracted text, supportingUrl is the result url it came from);
  @FunctionTool classifyDomain(List<SearchResult>) -> String (deterministic classification
  into one of: analytics, dataops, ml-platform, ai-app-framework, observability, other).

- PublishTools.java — @FunctionTool buildNotionPage(ProfileSummary) -> Map<String,Object>
  (assembles a Notion page properties payload from the ProfileSummary fields; keyed by Notion
  property name as declared in NotionSchema.FIELD_MAP); @FunctionTool writeToNotion(
  Map<String,Object> page) -> NotionPageRef (posts to the Notion pages API using
  NOTION_API_TOKEN targeting NOTION_DATABASE_ID; in mock mode writes to an in-process store
  and returns a synthetic NotionPageRef).

- NotionWriteGuardrail.java — implements the before-tool-call hook. Reads the candidate tool
  call's phase annotation from the registry and the CompetitorEntity status by competitorId
  (carried in the TaskDef metadata). Phase-gate check: same accept matrix as Section 8. For
  writeToNotion calls, additionally: (a) check that all NotionSchema.REQUIRED_FIELDS keys are
  present in the payload; (b) check that each value's runtime type matches NotionSchema.FIELD_TYPES;
  (c) check that the resolved NOTION_DATABASE_ID equals NotionSchema.ALLOWED_DATABASE_ID (read
  from env at startup). On any failure: return Guardrail.reject("schema-violation: ..." or
  "scope-violation: ..." or "phase-violation: ...") AND call
  CompetitorEntity.recordGuardrailRejection(phase, tool, reason).

- NotionSchema.java — constants: REQUIRED_FIELDS (Set<String> of the five required property
  names: "Name", "Pricing Model", "Primary Use Case", "Domain", "Source URL");
  FIELD_TYPES (Map<String,String> mapping each property name to its Notion type: "title",
  "rich_text", "select", "url"); ALLOWED_DATABASE_ID (String read from
  ${?NOTION_DATABASE_ID} at startup). Mock mode sets ALLOWED_DATABASE_ID = "mock-db-id".

- PublishQualityScorer.java — pure deterministic logic (no LLM). Inputs: CompetitorProfile,
  ProfileSummary, SearchResultSet. Outputs: EvalResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1: field coverage (all five
  REQUIRED_FIELDS dimensions have a non-empty ProfileField in the summary), source attribution
  (every ProfileField.supportingUrl is non-empty), source provenance (every
  ProfileField.supportingUrl appears in SearchResultSet.results[].url), and Notion ref
  present (profile.notionRef is non-null and pageUrl is non-empty). Score range 1-5. Rationale
  names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9901 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also reads ${?EXA_API_KEY},
  ${?NOTION_API_TOKEN}, ${?NOTION_DATABASE_ID}.

- src/main/resources/sample-events/competitors.jsonl with 5 seeded competitor names.

- src/main/resources/sample-data/competitors/*.json — three files keyed by seeded competitor
  name, each carrying 5-8 SearchResult entries with deterministic content so
  SearchTools.searchCompetitor returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in Section 8.
  Matching simplified_view list. No regulation_anchors — general domain.

- risk-survey.yaml at the project root with sector = research-intel, data.data_classes.pii =
  false (competitor names are public entities), decisions.authority_level = recommend-only
  (the profile is advisory), oversight.human_in_loop = true (an analyst reviews before acting),
  operations.agent_count = 1, operations.agent_pattern = sequential-pipeline, failure.failure_modes
  including "schema-violation-notion-write", "phase-violation", "missing-field-coverage",
  "hallucinated-source-url"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/research-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Competitor Research Pipeline",
  prerequisites including Exa.ai and Notion credentials, generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of competitor cards; right = selected-competitor detail with name header, search
  results table, profile summary fields, full profile, eval-score chip, rejection-log strip).
  Browser title exactly: <title>Akka Sample: Competitor Research Pipeline</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock. Also sets Exa.ai and Notion to mock mode.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write any key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(competitorId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    search-competitor.json — 5 SearchResultSet entries, each with 5-8 SearchResult items per
      seeded competitor. Each entry's tool_calls array contains 2-3 calls: 1
      searchCompetitor(name) + 1-2 fetchPage(url) calls. Plus 1 deliberately PHASE-VIOLATING
      entry whose tool_calls array starts with a writeToNotion(...) call (a PUBLISH-phase tool
      called during the SEARCH phase) — the guardrail rejects it, the mock then falls through
      to a normal search sequence. The mock selects the violating entry on the FIRST iteration
      of every 3rd competitor (modulo seed) so J2 is reproducible.
    summarize-findings.json — 5 ProfileSummary entries paired one-to-one with the search
      entries, each with all five required ProfileField dimensions, with tool_calls containing
      extractFields + classifyDomain in order.
    publish-profile.json — 5 CompetitorProfile entries paired one-to-one. Each carries
      NotionPageRef with a synthetic pageId and pageUrl, tool_calls containing buildNotionPage
      + writeToNotion. Plus 1 deliberately SCHEMA-VIOLATING entry whose buildNotionPage output
      omits "pricing_model" — the guardrail catches it with a schema-violation error; J3
      verifies this.
- A MockModelProvider.seedFor(competitorId) helper makes per-competitor selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ResearchAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ResearchTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (searchStep
  90s, summarizeStep 90s, publishStep 90s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on CompetitorRecord is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ResearchTasks.java with SEARCH_COMPETITOR, SUMMARIZE_FINDINGS, PUBLISH_PROFILE
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9901 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label is "Requires Exa.ai + Notion credentials (mock mode available)"
  — never T1/T2/T3/T4 in any user-visible string.
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write any key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ResearchAgent). The
  on-decision eval is rule-based (PublishQualityScorer.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  NotionWriteGuardrail is the runtime mechanism that enforces phase order and Notion schema/scope.
  Do NOT conditionally register tools per task — the guardrail is the gate.
- Task dependency is carried by typed task results: searchStep writes SearchResultSet onto the
  entity, summarizeStep reads it and builds the SUMMARIZE task's instruction context from it,
  publishStep reads both. The agent itself is stateless across phases.
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

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five valid options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
