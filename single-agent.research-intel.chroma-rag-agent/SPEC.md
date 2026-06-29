# SPEC — chroma-rag-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ChromaRagAgent.
**One-line pitch:** A user submits a natural-language question; one AI agent retrieves the top-k most relevant passages from a ChromaDB-backed corpus and returns a structured `RagAnswer` — response text plus a list of citations, each linking the claim to the chunk that supports it.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `RagAgent` (AutonomousAgent) carries the entire retrieval-and-answer decision; the surrounding components prepare the corpus index, manage the query lifecycle, and audit the agent's output. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates every candidate answer: every cited `chunkId` must appear in the retrieved context the agent was given, and the `citations` list must be non-empty when the answer is not a refusal. An answer that hallucinate a chunk ID or returns zero citations triggers a retry inside the same task.

The blueprint shows that a RAG agent is not inherently trustworthy because it has a retrieval step — grounding still needs to be enforced at the response boundary. One independent check sits on the answer before it reaches the caller.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** field (or picks one of three seeded examples — an Akka entity lifecycle question, a workflow steps question, and a component-topology question).
2. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `RETRIEVING` — the `QueryWorkflow` has called ChromaDB, found the top-k chunks, and passed them to the agent.
4. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWERED`. The answer appears: a response text paragraph, and a citations table (chunkId, document title, passage excerpt).
5. Within ~1 s of the answer, the `evalStep` finishes. The card shows a **groundedness score** chip (1–5) plus a one-line rationale describing how well the answer is anchored in the retrieved context.
6. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `RagView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → retrieving → answering → answered → evaluated. Source of truth. | `QueryEndpoint`, `QueryWorkflow` | `RagView` |
| `CorpusIndexer` | `Consumer` | Subscribes to `CorpusLoadRequested` events; drives ChromaDB embedding pipeline; emits `CorpusIndexed`. | `QueryEntity` events (corpus-init) | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `retrieveStep` → `answerStep` → `evalStep`. | started by `QueryEndpoint` after submit | `RagAgent`, `QueryEntity` |
| `RagAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question and the top-k retrieved chunks in the task definition; returns `RagAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `RagView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record RetrievedChunk(
    String chunkId,
    String documentTitle,
    String passage,
    double score          // cosine similarity 0..1
) {}

record QueryRequest(
    String queryId,
    String questionText,
    String submittedBy,
    Instant submittedAt
) {}

record Citation(
    String chunkId,
    String documentTitle,
    String passage        // verbatim passage from the retrieved chunk
) {}

record RagAnswer(
    String responseText,
    List<Citation> citations,
    int chunksRetrieved,
    Instant answeredAt
) {}

record GroundednessResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<List<RetrievedChunk>> retrievedChunks,
    Optional<RagAnswer> answer,
    Optional<GroundednessResult> groundedness,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, RETRIEVING, ANSWERING, ANSWERED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `ChunksRetrieved`, `AnsweringStarted`, `AnswerRecorded`, `GroundednessScored`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ questionText, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ChromaRagAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + groundedness score chip + age) and a right pane with the selected query's detail — question text, retrieved chunks list, answer text, and citations table.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `RagAgent`. Asserts the candidate response is well-formed `RagAnswer` JSON, every `citations[].chunkId` matches a `chunkId` present in the retrieved context that was passed to the agent, and `citations` is non-empty when `responseText` contains a factual claim (a refusal or "I don't know" answer may have zero citations). On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `RagAgent` → `prompts/rag-agent.md`. The single decision-making LLM. System prompt instructs it to read the retrieved chunks, compose an answer grounded in those chunks, and return one `Citation` per chunk it draws on.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks the seeded Akka-entities question; within 30 s the answer appears with at least one citation and a groundedness score chip.
2. **J2** — The agent's first response on a question cites a `chunkId` not in the retrieved context (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a grounded answer; the UI never displays the hallucinated citation.
3. **J3** — A question with no matching corpus chunks receives a groundedness score of 1 with a rationale noting that no retrieved passages support the response.
4. **J4** — Every citation in every recorded answer matches a `chunkId` from the `retrievedChunks` list on the same entity.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named chroma-rag-agent demonstrating the single-agent × research-intel cell. Runs
out of the box (no external services — ChromaDB embedded in-process). Maven group io.akka.samples.
Maven artifact single-agent-research-intel-chroma-rag-agent. Java package
io.akka.samples.ragusingchromadb. Akka 3.6.0. HTTP port 9535.

Components to wire (exactly):

- 1 AutonomousAgent RagAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/rag-agent.md>) and
  .capability(TaskAcceptance.of(RAG_TASKS.ANSWER_QUESTION).maxIterationsPerTask(3)). The task
  receives the user's question plus the top-k retrieved chunks (serialised as JSON in the task
  instructions field). Output: RagAnswer{responseText: String, citations: List<Citation>,
  chunksRetrieved: int, answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its 3-
  iteration budget.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * retrieveStep — calls ChromaDB (embedded client) to query the "docs" collection by the
    question text, using the configured embedding function, returning the top-5 chunks. Emits
    ChunksRetrieved, then advances to answerStep. WorkflowSettings.stepTimeout 20s.
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    RagAgent.class, "rag-" + queryId).runSingleTask(
      TaskDef.instructions(formatQuestionWithChunks(request.questionText, retrievedChunks))
    ) — returns a taskId, then forTask(taskId).result(RAG_TASKS.ANSWER_QUESTION) to fetch the
    answer. On success calls QueryEntity.recordAnswer(answer). WorkflowSettings.stepTimeout 60s
    with defaultStepRecovery maxRetries(2).failoverTo(QueryWorkflow::error).
  * evalStep — runs a deterministic rule-based GroundednessScorer (NOT an LLM call) over the
    recorded answer: checks that every Citation.chunkId appears in the retrievedChunks list,
    that citations is non-empty (if responseText is longer than 20 characters), and that no
    citation passage is a copy-paste of the responseText itself (a sign of fabrication).
    Emits GroundednessScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, retrievedChunks: Optional<List<RetrievedChunk>>,
  answer: Optional<RagAnswer>, groundedness: Optional<GroundednessResult>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED, RETRIEVING,
  ANSWERING, ANSWERED, EVALUATED, FAILED. Events: QuerySubmitted{request},
  ChunksRetrieved{chunks: List<RetrievedChunk>}, AnsweringStarted{}, AnswerRecorded{answer},
  GroundednessScored{groundedness}, QueryFailed{reason}. Commands: submit, markRetrieving,
  recordChunks, markAnswering, recordAnswer, recordGroundedness, fail, getQuery.
  emptyState() returns Query.initial("") with no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside the
  event-applier.

- 1 Consumer CorpusIndexer subscribed to QueryEntity events (corpus-init type); on
  CorpusLoadRequested drives the ChromaDB embedded client to create the "docs" collection, embed
  all passages from src/main/resources/corpus/docs-corpus.jsonl (sentence-transformer-like
  embedding function or a mock embedding function for the mock LLM path), and upsert chunks.
  On completion calls QueryEntity to emit CorpusIndexed. At service startup, the Bootstrap
  emits a single CorpusLoadRequested event to trigger this pipeline if the collection is empty.

- 1 View RagView with row type QueryRow (mirrors Query minus raw chunk passages — the view
  stores only chunkId + documentTitle per citation; full passage available via GET /api/queries/{id}).
  Table updater consumes QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM
  rag_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {questionText, submittedBy}; mints queryId;
    calls QueryEntity.submit; starts QueryWorkflow; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse (Server-
    Sent Events forwarded from the view's stream-updates), and three /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- RagTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer question")
  .description("Given the user question and retrieved chunks, produce a grounded RagAnswer")
  .resultConformsTo(RagAnswer.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records QueryRequest, RetrievedChunk, Citation, RagAnswer, GroundednessResult, Query,
  QueryStatus.

- CitationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  RagAnswer from the LLM response, runs the two checks listed in eval-matrix.yaml G1, and
  either passes the response through or returns Guardrail.reject(<structured-error>) to force
  the agent loop to retry.

- GroundednessScorer.java — pure deterministic logic (no LLM). Inputs: RagAnswer and the list
  of RetrievedChunk. Outputs: GroundednessResult. Scoring rubric documented in Javadoc on the
  class.

- ChromaDbClient.java — thin wrapper around the embedded ChromaDB Java client (dependency:
  io.github.njavet.chromadb:chromadb-java-client or equivalent in-process variant). Exposes
  query(collectionName, questionText, topK) returning List<RetrievedChunk>.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9535 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The RagAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/corpus/docs-corpus.jsonl with 30 seeded chunks from the Akka documentation
  (entity lifecycle, workflow steps, component topology, view queries, consumer patterns). Each
  chunk: { "chunkId": "akka-doc-NNN", "documentTitle": "...", "passage": "..." }.

- src/main/resources/sample-events/seed-questions.jsonl with 3 seeded questions paired with
  the expected top-chunk ids: an Akka entity lifecycle question, a workflow timeout question,
  and a view-query limitation question.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in Section 8
  of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (question text is not
  PII by default), decisions.authority_level = recommend-only (the agent's answer is informational,
  not enforceable), oversight.human_in_loop = false (retrieval-augmented answers are consumed
  directly by the user), failure.failure_modes including "hallucinated-citation", "out-of-corpus-claim",
  "top-k-retrieval-miss", "groundedness-score-suppressed"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/rag-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: RAG Using ChromaDB", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with question text, retrieved-chunks
  accordion, answer text, citations table, and groundedness score chip).
  Browser title exactly: <title>Akka Sample: ChromaRagAgent</title>. No subtitle on the
  Overview tab.

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
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 8 RagAnswer entries covering the three seeded questions and a
      general response. Each entry has a responseText paragraph and a citations list with
      chunkId values drawn from the seeded docs-corpus.jsonl. Severities vary. Plus 2
      deliberately MALFORMED entries (one citing a chunkId that does not appear in any
      retrieved context; one with an empty citations list despite a factual responseText)
      — the guardrail blocks both, exercising the retry path. The mock selects a malformed
      entry on the FIRST iteration of every 3rd query (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. RagAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RagTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (retrieveStep
  20s, answerStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>.
- Lesson 7: RagTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(RagAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9535 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: there is exactly ONE AutonomousAgent (RagAgent). The groundedness
  eval is rule-based (GroundednessScorer.java) and does NOT make an LLM call.
- The question and retrieved chunks are passed as task instructions text, not as a file
  attachment. The chunks are serialised as a JSON array in the instructions field so the
  guardrail can cross-reference chunkIds without a second retrieval.
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
