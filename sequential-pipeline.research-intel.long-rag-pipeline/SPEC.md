# SPEC — long-rag-workflow

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Long RAG Workflow.
**One-line pitch:** A user submits a research query; one `DocumentAgent` walks it through three task phases — **RETRIEVE** relevant chunks from long documents, **SYNTHESIZE** them into grounded findings and themes, **WRITE** a structured research report — with each phase gated on the prior phase's recorded output and long-context citation integrity enforced after every model response.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a research-intelligence domain tuned for very long documents. One `DocumentAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the RETRIEVE task's typed output becomes the SYNTHESIZE task's instruction context; the SYNTHESIZE task's typed output becomes the WRITE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

Two governance mechanisms are wired around the pipeline:

- An **`after-llm-response` guardrail** sits between the agent's model response and the task result that is returned to the workflow. After every model-generated passage, `CitationGuardrail` checks that each non-trivial sentence in the response cites at least one chunk id from the in-flight `ChunkWindow`. A response with uncited sentences is held; the agent receives a structured `citation-gap` instruction to revise and re-cite within its iteration budget. This is the long-document specific risk surface: with extended context, models can plausibly-sound confabulate details from documents that are semantically near but were not actually retrieved into the active window.
- An **`on-decision-eval`** runs immediately after `ReportWritten` lands, as `evalStep` inside the workflow. A deterministic, rule-based `CoverageScorer` (no LLM call — the eval is rule-based on purpose, so the same report always scores the same) checks that every synthesis theme has a matching report section, that every section cites at least two distinct chunks, that every cited chunk id appears in the recorded `ChunkWindow`, and that section count equals theme count.

The blueprint shows that in a long-document RAG pipeline, the greatest risk is not prompt injection or tool misuse — it is plausible-but-ungrounded generation. The `after-llm-response` hook is the right cut for that risk: inspect every response before it leaves the agent, not before a tool is called.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **research query** into the input (or picks one of three seeded queries — `EU AI Act liability provisions`, `NIST AI RMF alignment strategies`, `long-context model evaluation benchmarks`).
2. The user clicks **Run pipeline**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `RETRIEVING` — the workflow has started `retrieveStep` and the agent has been handed the RETRIEVE task.
4. Within ~15–30 s the card reaches `SYNTHESIZING` — the typed `ChunkWindow` is visible in the card detail (a small table of chunks with doc id, page range, and text preview). The agent's RETRIEVE task returned; the workflow recorded `ChunksRetrieved` and ran the SYNTHESIZE task.
5. Within ~15–30 s more the card reaches `REPORTING`. The `Synthesis` is visible (finding list + theme list with finding assignments).
6. Within ~15–30 s more the card reaches `EVALUATED`. The right pane now shows the full typed `ResearchReport` — title, abstract, per-theme `ReportSection` with body paragraphs and a `citations` list — plus a coverage score chip (1–5) and a one-line rationale.
7. The user can submit another query; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: created → retrieving → retrieved → synthesizing → synthesized → reporting → reported → evaluated. Source of truth. | `QueryEndpoint`, `LongRagPipelineWorkflow` | `QueryView` |
| `LongRagPipelineWorkflow` | `Workflow` | One workflow per query. Steps: `retrieveStep` → `synthesizeStep` → `reportStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `QueryEndpoint` after `CREATED` | `DocumentAgent`, `QueryEntity` |
| `DocumentAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `DocumentTasks.java`: `RETRIEVE_CHUNKS` → `ChunkWindow`, `SYNTHESIZE_FINDINGS` → `Synthesis`, `WRITE_REPORT` → `ResearchReport`. Each task is registered with the phase-appropriate function tools. | invoked by `LongRagPipelineWorkflow` | returns typed results |
| `RetrieveTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `searchChunks(query, maxChunks)` and `fetchChunkText(chunkId)`. Uses sliding-window chunking with configurable overlap. Reads from `src/main/resources/sample-data/corpus/*.json` for deterministic offline output. | called from RETRIEVE task | returns `List<Chunk>` |
| `SynthesizeTools` | function-tools class | Implements `extractFindings(chunks)` and `groupFindings(findings)`. Pure in-memory transformations that cluster by semantic similarity score. | called from SYNTHESIZE task | returns `List<Finding>` / `List<Theme>` |
| `ReportTools` | function-tools class | Implements `draftSection(theme, findings)` and `gatherCitations(findings)`. | called from REPORT task | returns `ReportSection` / `List<Citation>` |
| `CitationGuardrail` | `after-llm-response` guardrail (registered on `DocumentAgent`) | Reads the model's raw response text and the in-flight `ChunkWindow`. For every non-trivial sentence, verifies at least one `[chunkId]` citation tag is present. Responses failing the check receive a `citation-gap` revision instruction before being returned to the workflow. | every model response in every task | accept / structured-revise |
| `CoverageScorer` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `ResearchReport`, `Synthesis`, `ChunkWindow`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Chunk(
    String chunkId,
    String docId,
    String docTitle,
    int pageStart,
    int pageEnd,
    String text,
    Instant indexedAt
) {}

record ChunkWindow(
    List<Chunk> chunks,
    String queryText,
    Instant retrievedAt
) {}

record Finding(
    String findingId,
    String text,
    String supportingChunkId
) {}

record Theme(
    String themeId,
    String label,
    List<String> findingIds
) {}

record Synthesis(
    List<Theme> themes,
    List<Finding> findings,
    Instant synthesizedAt
) {}

record Citation(
    String chunkId,
    String docTitle,
    String pageRange
) {}

record ReportSection(
    String themeId,
    String heading,
    String body,
    List<Citation> citations
) {}

record ResearchReport(
    String title,
    String abstractText,
    List<ReportSection> sections,
    Instant writtenAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record QueryRecord(
    String queryId,
    Optional<String> queryText,
    Optional<ChunkWindow> chunkWindow,
    Optional<Synthesis> synthesis,
    Optional<ResearchReport> report,
    Optional<EvalResult> eval,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    CREATED, RETRIEVING, RETRIEVED, SYNTHESIZING, SYNTHESIZED,
    REPORTING, REPORTED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QueryCreated`, `RetrieveStarted`, `ChunksRetrieved`, `SynthesizeStarted`, `FindingsSynthesized`, `ReportStarted`, `ReportWritten`, `EvaluationScored`, `CitationFlagged`, `QueryFailed`.

Every nullable lifecycle field on the `QueryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ queryText }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Long RAG Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of queries (status pill + query text + age) and a right pane with the selected query's detail — query text, retrieved chunks table, synthesis themes/findings, report sections, coverage score chip, and a citation-flag log strip if any citation gaps were detected during generation.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — `after-llm-response` guardrail (citation-gate)**: `CitationGuardrail` is registered on `DocumentAgent` and runs after every model response. It scans the raw response text for `[chunkId]` citation tags and matches them against the set of chunk ids in the active `ChunkWindow` (carried in the `TaskDef` metadata as a JSON-encoded id list). The accept rule: every sentence of ≥ 10 words must contain at least one `[chunkId]` tag whose id appears in the `ChunkWindow`. On a citation gap, the guardrail returns a structured `citation-gap` revision instruction to the agent, naming the uncited sentence(s) and listing the available chunk ids for that window. The agent loop revises within its 5-iteration budget. On any revision, the guardrail records a `CitationFlagged{sentence, missingCitation, windowSize}` event on `QueryEntity` for audit. On accept, the response proceeds to the workflow.
- **E1 — `on-decision-eval`**: runs immediately after `ReportWritten` lands, as `evalStep` inside the workflow. `CoverageScorer` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest). Four checks, one point per check satisfied, starting from a base of 1: theme coverage (every synthesis theme has a matching section), citation density (every section cites ≥ 2 distinct chunk ids), chunk provenance (every cited `Citation.chunkId` appears in the recorded `ChunkWindow.chunks[].chunkId` list), and section parity (`sections.size() == themes.size()`). Emits `EvaluationScored{score:1..5, rationale}` with a rationale sentence pinpointing the largest gap. Reports with score ≤ 2 are flagged on the UI card.

## 9. Agent prompts

- `DocumentAgent` → `prompts/document-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, return the task's typed output, and tag every non-trivial sentence in synthesize/report responses with `[chunkId]` citations from the active window.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded query `EU AI Act liability provisions`; within 90 s the query reaches `EVALUATED` with non-empty chunks, ≥ 2 themes, ≥ 2 sections, and a coverage score chip on the card.
2. **J2** — The agent's first iteration on a query generates a passage with no citation tags (mock LLM path). `CitationGuardrail` holds the response; a `CitationFlagged` event lands on the entity; the agent revises with citations; the query eventually completes correctly. The UI's citation-flag log strip shows the one flagged passage.
3. **J3** — A query whose mock-LLM trajectory produces a section citing a chunk id absent from the recorded `ChunkWindow` is scored 1 with a rationale naming the out-of-window citation; the UI flags the card.
4. **J4** — Each task's instructions, attachments, and tool calls are visible in the per-query trace (logged at `INFO`); the RETRIEVE task's log shows only RETRIEVE-tool calls, the SYNTHESIZE task's log shows only SYNTHESIZE-tool calls, the REPORT task's log shows only REPORT-tool calls.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named long-rag-workflow demonstrating the sequential-pipeline x research-intel
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-research-intel-long-rag-pipeline. Java package
io.akka.samples.longragworkflow. Akka 3.6.0. HTTP port 9544.

Components to wire (exactly):

- 1 AutonomousAgent DocumentAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/document-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  5)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  RETRIEVE, SYNTHESIZE, and REPORT tool sets are ALL registered on the agent; phase gating
  is NOT the primary concern here (the after-llm-response guardrail is). The after-llm-response
  guardrail (CitationGuardrail) is registered on the agent via the agent's
  guardrail-configuration block. On a citation gap, the guardrail returns a citation-gap
  revision instruction to the agent loop; the loop retries within its 5-iteration budget.

- 1 Workflow LongRagPipelineWorkflow per queryId with four steps:
  * retrieveStep — emits RetrieveStarted on the entity, then calls componentClient
    .forAutonomousAgent(DocumentAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions("Query: " + queryText + "\nPhase: RETRIEVE\nUse the search and fetch
      tools to retrieve 6-12 relevant chunks. Prefer chunks that are semantically dense.")
        .metadata("queryId", queryId)
        .metadata("phase", "RETRIEVE")
        .taskType(DocumentTasks.RETRIEVE_CHUNKS)
    ). Reads forTask(taskId).result(RETRIEVE_CHUNKS) to get ChunkWindow. Writes
    QueryEntity.recordChunks(chunkWindow). WorkflowSettings.stepTimeout 90s.
  * synthesizeStep — emits SynthesizeStarted, then runSingleTask with TaskDef.instructions
    (formatSynthesizeContext(chunkWindow, queryText)) and metadata.phase = "SYNTHESIZE", taskType
    SYNTHESIZE_FINDINGS. Writes QueryEntity.recordSynthesis(synthesis). stepTimeout 90s.
  * reportStep — emits ReportStarted, then runSingleTask with TaskDef.instructions
    (formatReportContext(synthesis, chunkWindow, queryText)) and metadata.phase = "REPORT", taskType
    WRITE_REPORT. Writes QueryEntity.recordReport(report). stepTimeout 90s.
  * evalStep — runs the deterministic CoverageScorer over (report, synthesis, chunkWindow) and
    writes QueryEntity.recordEvaluation(eval). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(LongRagPipelineWorkflow::error). The error step writes
  QueryFailed and ends.

- 1 EventSourcedEntity QueryEntity (one per queryId). State QueryRecord{queryId,
  queryText: Optional<String>, chunkWindow: Optional<ChunkWindow>,
  synthesis: Optional<Synthesis>, report: Optional<ResearchReport>,
  eval: Optional<EvalResult>, status: QueryStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. QueryStatus enum: CREATED, RETRIEVING, RETRIEVED,
  SYNTHESIZING, SYNTHESIZED, REPORTING, REPORTED, EVALUATED, FAILED. Events:
  QueryCreated{queryText}, RetrieveStarted, ChunksRetrieved{chunkWindow}, SynthesizeStarted,
  FindingsSynthesized{synthesis}, ReportStarted, ReportWritten{report},
  EvaluationScored{eval}, CitationFlagged{sentence, missingCitation, windowSize},
  QueryFailed{reason}.
  Commands: create, startRetrieve, recordChunks, startSynthesize, recordSynthesis,
  startReport, recordReport, recordEvaluation, recordCitationFlag, fail, getQuery.
  emptyState() returns QueryRecord.initial("") with all Optional fields as Optional.empty()
  and no commandContext() reference (Lesson 3).

- 1 View QueryView with row type QueryRow that mirrors QueryRecord exactly (all Optional<T>
  lifecycle fields preserved). Table updater consumes QueryEntity events. ONE query
  getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {queryText}; mints queryId; calls
    QueryEntity.create(queryText); then starts LongRagPipelineWorkflow with id
    "rag-" + queryId; returns {queryId}), GET /queries (list from getAllQueries,
    sorted newest-first), GET /queries/{id} (one row), GET /queries/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- DocumentTasks.java declaring three Task<R> constants:
    RETRIEVE_CHUNKS = Task.name("Retrieve chunks").description("Search the document corpus
      for relevant chunks using sliding-window retrieval, then fetch their full text.")
      .resultConformsTo(ChunkWindow.class);
    SYNTHESIZE_FINDINGS = Task.name("Synthesize findings").description("Extract factual
      findings from chunks, then group findings into themes.")
      .resultConformsTo(Synthesis.class);
    WRITE_REPORT = Task.name("Write report").description("Compose a ResearchReport whose
      ReportSections mirror the Synthesis themes one-to-one, citing chunk ids.")
      .resultConformsTo(ResearchReport.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- RetrieveTools.java — @FunctionTool searchChunks(String query, int maxChunks) -> List<Chunk>
  reading from src/main/resources/sample-data/corpus/*.json using sliding-window chunking
  with 20% overlap; @FunctionTool fetchChunkText(String chunkId) -> String reading the
  full chunk text from the matching corpus entry.

- SynthesizeTools.java — @FunctionTool extractFindings(List<Chunk>) -> List<Finding> (one
  Finding per Chunk, with findingId minted as "f-" + sha1(text).substring(0,8));
  @FunctionTool groupFindings(List<Finding>) -> List<Theme> (deterministic clustering —
  assign each Finding to a Theme keyed by the first significant noun phrase; 2-5 themes typical).

- ReportTools.java — @FunctionTool draftSection(Theme, List<Finding>) -> ReportSection
  (heading from theme label, body composed from finding texts with [chunkId] citation tags);
  @FunctionTool gatherCitations(List<Finding>) -> List<Citation> (one Citation per
  distinct supporting chunk).

- CitationGuardrail.java — implements the after-llm-response hook. Parses the model
  response text for [chunkId] citation tags. For every sentence of >= 10 words, verifies
  at least one [chunkId] tag whose id appears in the active ChunkWindow id set (carried in
  TaskDef.metadata as "chunkIds" JSON). On a citation gap, returns
  Guardrail.revise("citation-gap: the following sentences lack grounded citations: <list>.
  Available chunk ids for this window: <ids>. Please revise and add [chunkId] tags.")
  On any revision, ALSO calls QueryEntity.recordCitationFlag(sentence, missingCitation,
  windowSize) so the flag is visible in the UI's citation-flag log strip and in the
  audit log. On accept, the response proceeds.

- CoverageScorer.java — pure deterministic logic (no LLM). Inputs: ResearchReport, Synthesis,
  ChunkWindow. Outputs: EvalResult with score and rationale. Four checks, one point per check
  satisfied, starting from a base of 1: theme coverage (every Theme has a ReportSection),
  citation density (every ReportSection has >= 2 distinct Citation chunkIds), chunk provenance
  (every Citation.chunkId appears in ChunkWindow.chunks[].chunkId), and section parity
  (sections.size() == themes.size()). Score range 1-5.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9544 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/queries.jsonl with 5 seeded query lines covering the three
  surfaces named in J1-J5 plus two extras.

- src/main/resources/sample-data/corpus/*.json — three files keyed by seeded query domain,
  each carrying 10-15 Chunk entries (docId, docTitle, pageStart, pageEnd, text) with
  deterministic content so RetrieveTools.searchChunks returns the same list across restarts.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, E1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false, decisions.
  authority_level = recommend-only, oversight.human_in_loop = true, operations.agent_count
  = 1, operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "uncited-passage", "out-of-window-citation", "hallucinated-chunk", "thin-coverage-section",
  "chunk-boundary-split"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/document-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Long RAG Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with query text header, chunks
  table, synthesis themes/findings, report sections, coverage-score chip, citation-flag strip).
  Browser title exactly: <title>Akka Sample: Long RAG Workflow</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences.
- Per-task mock-response shapes:
    retrieve-chunks.json — 6 ChunkWindow entries, each with 8-12 Chunk items. Each entry's
      tool_calls array contains 2-4 calls: 1 searchChunks(query, 10) + 1-3 fetchChunkText
      calls. Plus 1 deliberately UNCITED entry whose model response text contains no
      [chunkId] tags — the citation guardrail flags it, the mock then falls through to a
      cited response. The mock selects the uncited entry on the FIRST iteration of every
      3rd query (modulo seed) so J2 is reproducible.
    synthesize-findings.json — 6 Synthesis entries paired one-to-one with the retrieve
      entries, each with 2-4 Theme items and 6-10 Finding items.
    write-report.json — 6 ResearchReport entries paired one-to-one. Each carries 2-4
      ReportSection items. Plus 1 deliberately OUT-OF-WINDOW entry whose first ReportSection
      cites a chunkId absent from the paired ChunkWindow — evalStep scores it 1; J3 verifies.
- MockModelProvider.seedFor(queryId) makes selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. DocumentAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion DocumentTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (retrieveStep
  90s, synthesizeStep 90s, reportStep 90s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the QueryRecord row record is Optional<T>.
- Lesson 7: DocumentTasks.java with RETRIEVE_CHUNKS, SYNTHESIZE_FINDINGS, WRITE_REPORT
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9544 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (DocumentAgent). CoverageScorer
  is rule-based and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's typed result is the dependency handoff to
  the next task. The agent is stateless across phases.
- The after-llm-response guardrail is the citation gate: it runs on the model response, not
  on tool calls. Tool calls are unrestricted — the citation check is the runtime enforcement.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
