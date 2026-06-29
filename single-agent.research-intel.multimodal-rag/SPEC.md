# SPEC — multiformat-hybrid-rag

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MultiformatHybridRAG.
**One-line pitch:** A user asks a natural-language question; one AI agent retrieves relevant passages from a mixed-format corpus (plain text, markdown, PDF-page text, image-derived captions), synthesises an answer, and returns a structured `ResearchAnswer` — every factual claim anchored to a cited passage id.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `ResearchAgent` (AutonomousAgent) owns the entire answer; the surrounding components prepare its retrieval input and enforce citation coverage on its output. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates the agent's answer on every turn: every factual sentence must carry at least one `citationId` that references a chunk in the retrieved set; the `ResearchAnswer` must be well-formed JSON; and when the agent asserts `NO_RESULT`, no spurious `citations` array may accompany it. A failing response triggers a retry inside the same task.

The blueprint shows that a single retrieval-augmented agent still benefits from a structural output check. Without the guardrail, an answer could assert facts with no passage backing — the model's generation quality is not a substitute for a deterministic coverage check at the boundary.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** textarea (or picks one of four seeded questions covering climate science, renewable energy policy, ocean biodiversity, and carbon capture technology).
2. The user clicks **Submit question**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `RETRIEVING` — the retriever has looked up the top-K chunks and attached them to the query record.
4. Within ~10–30 s, the `answerStep` completes. The card transitions to `ANSWERING` then `ANSWERED`. The answer appears: a decision badge (`ANSWERED` / `NO_RESULT`), a narrative answer paragraph, and a citation table (chunk id, source format, title, a short passage excerpt).
5. The user can submit another question; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → retrieving → answering → answered. Source of truth. | `QueryEndpoint`, `ChunkRetriever`, `QueryWorkflow` | `QueryView` |
| `ChunkIndexer` | `Consumer` | Subscribes to `CorpusChunkAdded` events; extracts text per format; writes indexed `ChunkRecord`. | `CorpusChunkAdded` events | `ChunkStore` (in-process map) |
| `ChunkStore` | supporting class | In-memory inverted-index over `ChunkRecord` entries; exposes `topK(question, k)`. | loaded from corpus JSONL at startup | `QueryWorkflow` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `retrieveStep` → `answerStep`. | started by `QueryEndpoint` | `ResearchAgent`, `QueryEntity` |
| `ResearchAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question as `TaskDef.instructions(...)` and up to 20 retrieved chunks as task attachments (one attachment per chunk); returns `ResearchAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `CitationGuardrail` | guardrail | `before-agent-response` hook on `ResearchAgent`. Asserts every cited chunk id exists in the retrieved set and every factual sentence carries a citation. | bound to `ResearchAgent` | returns accept or reject |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum SourceFormat { TEXT, MARKDOWN, PDF_PAGE, IMAGE_CAPTION }

record ChunkRecord(
    String chunkId,
    String sourceTitle,
    String sourceUri,
    SourceFormat format,
    String content,           // normalised plaintext ready for retrieval
    String rawContent,        // original before normalisation (stored for citation excerpt)
    int pageOrSequence        // page number (PDF) or insertion order (others)
) {}

record RetrievedChunk(
    String chunkId,
    String sourceTitle,
    SourceFormat format,
    String excerpt,           // first 300 chars of content
    double relevanceScore     // 0..1, keyword-overlap heuristic (no embedding service)
) {}

record Citation(
    String chunkId,
    String sourceTitle,
    SourceFormat format,
    String passageExcerpt
) {}

record ResearchAnswer(
    AnswerDecision decision,
    String answerText,        // full narrative, null when decision == NO_RESULT
    String noResultReason,    // populated when decision == NO_RESULT, null otherwise
    List<Citation> citations,
    Instant answeredAt
) {}
enum AnswerDecision { ANSWERED, NO_RESULT }

record QueryRequest(
    String queryId,
    String question,
    String submittedBy,
    Instant submittedAt
) {}

record RetrievalResult(
    List<RetrievedChunk> chunks,
    int totalChunksSearched,
    Instant retrievedAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<RetrievalResult> retrieval,
    Optional<ResearchAnswer> answer,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, RETRIEVING, ANSWERING, ANSWERED, NO_RESULT, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `RetrievalCompleted`, `AnsweringStarted`, `AnswerRecorded`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MultiformatHybridRAG</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + decision badge + age) and a right pane with the selected query's detail — submitted question, retrieved chunk list with format badges, answer narrative, and citation table.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ResearchAgent`. Asserts the candidate response is well-formed `ResearchAnswer` JSON; when `decision == ANSWERED`, asserts every `citations[].chunkId` is present in the retrieved chunk set for that query; asserts at minimum one citation exists when an `answerText` is provided; and asserts that if `decision == NO_RESULT` the `citations` list is empty. On failure returns a structured rejection naming which check failed; the agent loop consumes one of its 3 iterations and retries. Passing responses flow through to `QueryEntity.recordAnswer`.

## 9. Agent prompts

- `ResearchAgent` → `prompts/research-agent.md`. The single decision-making LLM. System prompt instructs it to read attached chunks, synthesise an answer that cites every chunk it draws from, and return `NO_RESULT` with a rationale when no chunk is relevant.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seeded climate-science question; within 30 s the answer appears with citations that reference at least 2 retrieved chunk ids and an `ANSWERED` decision badge.
2. **J2** — The agent's first response on a query is citation-free (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration includes citations; the UI shows only the final answer.
3. **J3** — A question matching image-caption chunks returns citations with `sourceFormat: IMAGE_CAPTION`; those citations are visually distinguished in the citation table.
4. **J4** — A question outside the seeded corpus scope returns `NO_RESULT` with a non-empty `noResultReason` and an empty `citations` array.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multiformathybridrag demonstrating the single-agent × research-intel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-multimodal-rag. Java package
io.akka.samples.multiformathybridrag. Akka 3.6.0. HTTP port 9969.

Components to wire (exactly):

- 1 AutonomousAgent ResearchAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/research-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUESTION).maxIterationsPerTask(3)). The task receives
  the user's question as its instruction text and up to 20 RetrievedChunk entries as task
  ATTACHMENTS (one attachment per chunk named "chunk-<chunkId>.txt" containing the chunk's
  content) — NOT as inline prompt text. Output: ResearchAnswer{decision: AnswerDecision
  (ANSWERED/NO_RESULT), answerText: String (nullable when NO_RESULT), noResultReason: String
  (nullable when ANSWERED), citations: List<Citation>, answeredAt: Instant}. The agent is
  configured with a before-agent-response guardrail (see G1 in eval-matrix.yaml) registered
  via the agent's guardrail-configuration block. On guardrail rejection the agent loop retries
  within its 3-iteration budget.

- 1 Workflow QueryWorkflow per queryId with two steps:
  * retrieveStep — calls ChunkStore.topK(request.question, 20) (synchronous in-process call,
    no IO); emits RetrievalCompleted with the ranked chunk list; calls
    QueryEntity.attachRetrieval(result). WorkflowSettings.stepTimeout 5s.
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    ResearchAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions(request.question)
        .attachments(buildAttachments(retrieval.chunks))
    ) — returns a taskId, then forTask(taskId).result(ANSWER_QUESTION) to fetch the answer.
    On success calls QueryEntity.recordAnswer(answer). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, retrieval: Optional<RetrievalResult>,
  answer: Optional<ResearchAnswer>, status: QueryStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED, RETRIEVING, ANSWERING,
  ANSWERED, NO_RESULT, FAILED. Events: QuerySubmitted{request}, RetrievalCompleted{retrieval},
  AnsweringStarted{}, AnswerRecorded{answer}, QueryFailed{reason}. Commands: submit,
  attachRetrieval, markAnswering, recordAnswer, fail, getQuery. emptyState() returns
  Query.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer ChunkIndexer subscribed to CorpusChunkAdded events. On each event, determines
  SourceFormat from the event's format field and extracts normalised plaintext: TEXT/MARKDOWN
  pass through (markdown strips headings/bullet markers), PDF_PAGE strips page-header tokens,
  IMAGE_CAPTION prefixes "Caption: ". Stores a ChunkRecord in ChunkStore. The ChunkStore is
  a singleton supporting class with a topK(question, k) method that scores chunks by
  keyword-overlap (word intersection / union, no embedding call) and returns the top k by
  relevanceScore. ChunkStore is loaded from
  src/main/resources/sample-events/corpus-chunks.jsonl at Bootstrap time by firing one
  CorpusChunkAdded command per entry into ChunkIndexer's subscribe path.

- 1 View QueryView with row type QueryRow (mirrors Query minus any large raw content fields
  not needed for the list view). Table updater consumes QueryEntity events. ONE query
  getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question, submittedBy}; mints queryId;
    calls QueryEntity.submit; starts QueryWorkflow; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer question")
  .description("Read the attached corpus chunks and return a ResearchAnswer with citations")
  .resultConformsTo(ResearchAnswer.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records: ChunkRecord, RetrievedChunk, Citation, ResearchAnswer, AnswerDecision,
  QueryRequest, RetrievalResult, Query, QueryStatus, SourceFormat.

- CitationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ResearchAnswer from the LLM response, runs the four checks listed in eval-matrix.yaml G1,
  and either passes the response through or returns Guardrail.reject(<structured-error>) to
  force the agent loop to retry.

- ChunkStore.java — pure in-memory retrieval (no LLM, no embedding). topK uses Jaccard
  coefficient over word sets (stop-word filtered). Populated by ChunkIndexer. Exposes
  topK(String question, int k) returning List<RetrievedChunk> sorted descending by
  relevanceScore. Thread-safe via ConcurrentHashMap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9969 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ResearchAgent.definition() binds
  the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/corpus-chunks.jsonl with 40 seeded ChunkRecord entries
  covering four domains (climate science, renewable energy policy, ocean biodiversity, carbon
  capture technology), 10 chunks each. Formats distributed: 16 TEXT, 12 MARKDOWN, 8 PDF_PAGE,
  4 IMAGE_CAPTION. Each chunk: 150–400 words of synthetic but plausible content. Each
  IMAGE_CAPTION chunk: 40–80-word caption describing a chart or photograph.

- src/main/resources/sample-events/seeded-questions.jsonl with 4 paired questions, one per
  domain, and an expected-citation-count field for acceptance testing.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false (questions are
  research queries, not personal data), decisions.authority_level = informational
  (the agent's answer informs the user; no automated action is taken), oversight.human_in_loop
  = true, failure.failure_modes including "hallucinated-citation", "unretrieved-claim",
  "no-result-suppressed", "format-misattribution"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/research-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: MultiformatHybridRAG", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with submitted question, retrieved
  chunk list with format badges, answer narrative, and citation table with source-format
  distinguishing icons).
  Browser title exactly: <title>Akka Sample: MultiformatHybridRAG</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; gone when the
        session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)).
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 6 ResearchAnswer entries:
      4 ANSWERED entries, one per seeded domain, each with 2–3 Citation objects whose
      chunkIds reference actual chunks in corpus-chunks.jsonl, a 2–4 sentence answerText,
      and a null noResultReason. Citations carry non-empty passageExcerpts.
      1 NO_RESULT entry with a non-empty noResultReason and an empty citations list (for J4).
      1 deliberately MALFORMED entry (a citation chunkId that is not in the retrieved set)
      — the guardrail blocks it, exercising the retry path. The mock selects the malformed
      entry on the FIRST iteration of every 3rd query (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ResearchAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (retrieveStep 5s, answerStep 60s,
  error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: QueryTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(ResearchAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9969 declared explicitly in application.conf's
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
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ResearchAgent). The
  ChunkStore retrieval is keyword-overlap heuristic (no LLM call, no embedding service).
  The citation guardrail is deterministic. Only ResearchAgent talks to a model.
- Chunks are passed as task ATTACHMENTS (one file per chunk), never inlined as instructions
  text. Verify the generated answerStep uses TaskDef.attachments(...) and not string
  interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
