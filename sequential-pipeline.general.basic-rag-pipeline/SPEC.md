# SPEC — basic-rag-pipeline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Basic RAG Workflow.
**One-line pitch:** A user submits a question; one `RagAgent` walks it through two task phases — **INGEST** a document corpus into an in-process vector store, **QUERY** the index with the user's question to produce a grounded `RagAnswer` — with the final answer filtered by a `before-agent-response` guardrail that blocks any response whose citations do not trace back to indexed chunks.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a general-purpose retrieval-augmented-generation domain. One `RagAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the INGEST task's typed output (`IndexedCorpus`) becomes the QUERY task's instruction context — the agent never holds both phases in one conversation, and the workflow is the only path information travels between them.

One governance mechanism is wired around the pipeline:

- A **`before-agent-response` guardrail** sits between the agent and the caller on the QUERY task's final answer. Before the `RagAnswer` is returned to `RagPipelineWorkflow`, the guardrail inspects every citation URL in the answer against the `IndexedCorpus.chunks[].sourceUrl` list recorded on `RagSessionEntity`. Any citation whose URL is absent from the indexed corpus is evidence of hallucination; the guardrail blocks the answer and returns a structured rejection to the agent loop. The agent may retry within its 4-iteration budget. On final block (budget exhausted), the workflow records `AnswerBlocked` and the session surfaces an honest failure to the user instead of a fabricated answer.

The blueprint shows that the `before-agent-response` hook is the right cut for output-level citation integrity: it runs after the agent has composed its full answer but before that answer becomes visible to any caller, giving the agent one chance to self-correct.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **question** into the input (or picks one of three seeded questions — `What are the key principles of transformer architectures?`, `How does retrieval-augmented generation differ from fine-tuning?`, `What is vector similarity search and when should I use it?`).
2. The user clicks **Ask**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `INDEXING` — the workflow has started `ingestStep` and the agent has been handed the INGEST task.
4. Within ~10–20 s the card reaches `INDEXED` — the `IndexedCorpus` is visible in the card detail (chunk count, document names). The agent's INGEST task returned; the workflow recorded `CorpusIndexed` and started the QUERY task.
5. Within ~10–20 s more the card reaches `ANSWERING`.
6. Within ~10–20 s more the card reaches `ANSWERED`. The right pane now shows the full `RagAnswer` — the question, the answer text, and a citations list with one entry per referenced chunk — plus a guardrail chip (PASSED or BLOCKED).
7. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RagEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `RagSessionEntity`, `RagSessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `RagSessionEntity` | `EventSourcedEntity` | Per-session lifecycle: created → indexing → indexed → answering → answered → blocked. Source of truth. | `RagEndpoint`, `RagPipelineWorkflow` | `RagSessionView` |
| `RagPipelineWorkflow` | `Workflow` | One workflow per session. Steps: `ingestStep` → `queryStep` → `guardrailStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `RagEndpoint` after `CREATED` | `RagAgent`, `RagSessionEntity` |
| `RagAgent` | `AutonomousAgent` | The single agent. Declares two `Task<R>` constants in `RagTasks.java`: `INGEST_CORPUS` → `IndexedCorpus`, `ANSWER_QUERY` → `RagAnswer`. Each task is registered with the phase-appropriate function tools. | invoked by `RagPipelineWorkflow` | returns typed results |
| `IngestTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `loadDocuments(corpusId)` and `indexChunks(documents)`. Reads from `src/main/resources/sample-data/corpus/*.json` for deterministic offline output; writes chunks into an in-process `VectorStore`. | called from INGEST task | returns `List<IndexedChunk>` |
| `QueryTools` | function-tools class | Implements `retrieveChunks(question, topK)` and `buildContext(chunks)`. Reads from the in-process `VectorStore`. | called from QUERY task | returns `List<IndexedChunk>` / `String` |
| `VectorStore` | plain class (singleton) | In-process keyword-similarity index over `IndexedChunk` records. Deterministic — same corpus, same query, same ranked results. | populated by `IngestTools.indexChunks` | read by `QueryTools.retrieveChunks` |
| `AnswerGuardrail` | `before-agent-response` guardrail (registered on `RagAgent`) | Inspects every `RagAnswer.citations[].sourceUrl` against the `IndexedCorpus.chunks[].sourceUrl` list on `RagSessionEntity`. Blocks answers with uncorroborated citations. Records a `GuardrailApplied` event for audit. | every final answer on ANSWER_QUERY task | accept / structured-reject |
| `RagSessionView` | `View` | Read model: one row per session for the UI. | `RagSessionEntity` events | `RagEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record DocumentSource(String docId, String title, String content, String sourceUrl) {}

record IndexedChunk(
    String chunkId,
    String docId,
    String text,
    String sourceUrl,
    int chunkIndex
) {}

record IndexedCorpus(
    String corpusId,
    List<IndexedChunk> chunks,
    Instant indexedAt
) {}

record Citation(
    String chunkId,
    String sourceUrl,
    String excerpt
) {}

record RagAnswer(
    String question,
    String answerText,
    List<Citation> citations,
    Instant answeredAt
) {}

record GuardrailOutcome(
    String verdict,      // "PASSED" or "BLOCKED"
    String reason,
    Instant evaluatedAt
) {}

record RagSessionRecord(
    String sessionId,
    Optional<String> question,
    Optional<IndexedCorpus> corpus,
    Optional<RagAnswer> answer,
    Optional<GuardrailOutcome> guardrailOutcome,
    RagSessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum RagSessionStatus {
    CREATED, INDEXING, INDEXED, ANSWERING, ANSWERED, BLOCKED, FAILED
}
```

Events on `RagSessionEntity`: `SessionCreated`, `IngestStarted`, `CorpusIndexed`, `QueryStarted`, `AnswerDrafted`, `GuardrailApplied`, `AnswerBlocked`, `SessionFailed`.

Every nullable lifecycle field on the `RagSessionRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ question }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Basic RAG Workflow</title>`.

The App UI tab is a two-column layout: a left rail with the live list of sessions (status pill + question excerpt + age) and a right pane with the selected session's detail — question, indexed-corpus chunk count, answer text with inline citation chips, and a guardrail-outcome chip (PASSED or BLOCKED with reason).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — `before-agent-response` guardrail (citation check)**: `AnswerGuardrail` is registered on `RagAgent` and runs before the ANSWER_QUERY task's final response is returned to the workflow. It reads every `RagAnswer.citations[].sourceUrl` and checks each against `IndexedCorpus.chunks[].sourceUrl` recorded on the session's `RagSessionEntity`. The accept rule: every citation URL must appear in the indexed corpus. On reject, the guardrail returns a structured `citation-not-grounded` error to the agent loop and the workflow records a `GuardrailApplied{verdict: "BLOCKED", reason}` event. The agent may retry within its 4-iteration budget; if all iterations produce ungrounded citations, the workflow records `AnswerBlocked` and the session transitions to `BLOCKED`. On accept, the workflow records `GuardrailApplied{verdict: "PASSED"}` and the session transitions to `ANSWERED`.

## 9. Agent prompts

- `RagAgent` → `prompts/rag-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools from the matching phase's tool set, and — critically — never cite a URL that was not returned by `retrieveChunks`. The `before-agent-response` guardrail enforces the citation rule at runtime; the system prompt plants the behavioral expectation.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded question `What are the key principles of transformer architectures?`; within 45 s the session reaches `ANSWERED` with a non-empty answer, ≥ 1 citation, and a `PASSED` guardrail chip.
2. **J2** — The mock LLM produces an answer whose first citation references a URL absent from the indexed corpus. `AnswerGuardrail` blocks it; a `GuardrailApplied{verdict: BLOCKED}` event lands on the entity; the agent retries; the session eventually reaches `ANSWERED` with a grounded answer. The guardrail chip reads `PASSED` on the final answer.
3. **J3** — A question with no semantically relevant chunks (e.g., `What is the GDP of the moon?`) reaches the QUERY phase but `retrieveChunks` returns an empty list. The agent returns a `RagAnswer` with `answerText = "No relevant content found in the indexed corpus."` and `citations = []`. The guardrail passes (no citations to invalidate). The session reaches `ANSWERED` honestly.
4. **J4** — Each task's tool calls are visible in the per-session trace (logged at `INFO`); the INGEST task's log shows only `loadDocuments` and `indexChunks` calls; the QUERY task's log shows only `retrieveChunks` and `buildContext` calls. The trace is empirical evidence the dependency contract holds end-to-end.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named basic-rag-pipeline demonstrating the sequential-pipeline x general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
sequential-pipeline-general-basic-rag-pipeline. Java package io.akka.samples.basicragworkflow.
Akka 3.6.0. HTTP port 9886.

Components to wire (exactly):

- 1 AutonomousAgent RagAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/rag-agent.md>) and two .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(4))
  entries — one per declared Task. Function tools are registered with .tools(...) — both
  INGEST and QUERY tool sets are ALL registered on the agent; phase awareness is communicated
  via the task's instruction context and the before-agent-response guardrail. The
  before-agent-response guardrail (AnswerGuardrail) is registered on the agent via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries within
  its 4-iteration budget.

- 1 Workflow RagPipelineWorkflow per sessionId with three steps:
  * ingestStep — emits IngestStarted on the entity, then calls componentClient
    .forAutonomousAgent(RagAgent.class, "agent-" + sessionId).runSingleTask(
      TaskDef.instructions("CorpusId: default\nPhase: INGEST\nLoad and index all documents
      in the default corpus.")
        .metadata("sessionId", sessionId)
        .metadata("phase", "INGEST")
        .taskType(RagTasks.INGEST_CORPUS)
    ). Reads forTask(taskId).result(INGEST_CORPUS) to get IndexedCorpus. Writes
    RagSessionEntity.recordCorpus(corpus). WorkflowSettings.stepTimeout 60s.
  * queryStep — emits QueryStarted, then runSingleTask with TaskDef.instructions
    (formatQueryContext(corpus, question)) and metadata.phase = "QUERY", taskType
    ANSWER_QUERY. The before-agent-response guardrail fires here before the result
    is returned to the workflow. Reads forTask(taskId).result(ANSWER_QUERY) to get
    RagAnswer. Writes RagSessionEntity.recordAnswer(answer, guardrailOutcome). stepTimeout 60s.
  * error step — writes SessionFailed and ends. stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(RagPipelineWorkflow::error). The error step writes
  SessionFailed and ends.

- 1 EventSourcedEntity RagSessionEntity (one per sessionId). State RagSessionRecord{sessionId,
  question: Optional<String>, corpus: Optional<IndexedCorpus>, answer: Optional<RagAnswer>,
  guardrailOutcome: Optional<GuardrailOutcome>, status: RagSessionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. RagSessionStatus enum: CREATED, INDEXING, INDEXED,
  ANSWERING, ANSWERED, BLOCKED, FAILED. Events: SessionCreated{question}, IngestStarted,
  CorpusIndexed{corpus}, QueryStarted, AnswerDrafted{answer}, GuardrailApplied{verdict,
  reason}, AnswerBlocked{reason}, SessionFailed{reason}. Commands: create, startIngest,
  recordCorpus, startQuery, recordAnswer, recordGuardrailOutcome, blockAnswer, fail,
  getSession. emptyState() returns RagSessionRecord.initial("") with all Optional fields as
  Optional.empty() and no commandContext() reference (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View RagSessionView with row type RagSessionRow that mirrors RagSessionRecord exactly
  (all Optional<T> lifecycle fields preserved). Table updater consumes RagSessionEntity events.
  ONE query getAllSessions: SELECT * AS sessions FROM rag_session_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * RagEndpoint at /api with POST /sessions (body {question}; mints sessionId; calls
    RagSessionEntity.create(question); then starts RagPipelineWorkflow with id
    "rag-" + sessionId; returns {sessionId}), GET /sessions (list from getAllSessions, sorted
    newest-first), GET /sessions/{id} (one row), GET /sessions/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- RagTasks.java declaring two Task<R> constants:
    INGEST_CORPUS = Task.name("Ingest corpus").description("Load documents from the default
      corpus and index them into the in-process vector store").resultConformsTo(
      IndexedCorpus.class);
    ANSWER_QUERY = Task.name("Answer query").description("Retrieve relevant chunks for the
      question and compose a grounded answer with citations").resultConformsTo(
      RagAnswer.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- IngestTools.java — @FunctionTool loadDocuments(String corpusId) -> List<DocumentSource>
  reading from src/main/resources/sample-data/corpus/*.json; @FunctionTool indexChunks(
  List<DocumentSource> documents) -> List<IndexedChunk> splitting each document body into
  chunks of ~200 words and writing them into VectorStore.

- QueryTools.java — @FunctionTool retrieveChunks(String question, int topK) ->
  List<IndexedChunk> doing keyword-similarity lookup against VectorStore; @FunctionTool
  buildContext(List<IndexedChunk> chunks) -> String concatenating chunk texts with source
  labels, truncated to 2000 chars.

- VectorStore.java — singleton plain class. Stores List<IndexedChunk>. retrieveChunks does
  keyword overlap scoring: tokenise question, score each chunk by count of matching tokens,
  return top-K by score. Deterministic given the same corpus and question.

- AnswerGuardrail.java — implements the before-agent-response hook. Reads every
  RagAnswer.citations[].sourceUrl and checks each against the IndexedCorpus.chunks[].sourceUrl
  list looked up from RagSessionEntity by sessionId (carried in the TaskDef metadata). If all
  citations are grounded: returns Guardrail.accept() and calls
  RagSessionEntity.recordGuardrailOutcome("PASSED", "All citations grounded"). If any citation
  is missing: returns Guardrail.reject("citation-not-grounded: <url> not in indexed corpus")
  and calls RagSessionEntity.recordGuardrailOutcome("BLOCKED", reason).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9886 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-data/corpus/*.json — three JSON files, each carrying a list of
  DocumentSource records. Documents cover: (1) transformer-architecture.json — 4 documents
  on self-attention, positional encoding, encoder-decoder stacks, and multi-head attention;
  (2) rag-vs-finetuning.json — 3 documents on retrieval-augmented generation, parameter
  updates, and knowledge freshness; (3) vector-search.json — 3 documents on cosine similarity,
  approximate-nearest-neighbour algorithms, and embedding spaces. Each document carries a
  non-null sourceUrl, title, docId, and content of ≥ 3 sentences.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors — general
  domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (questions are
  topical, not personal), decisions.authority_level = recommend-only (the answer is advisory),
  oversight.human_in_loop = true, operations.agent_count = 1,
  operations.agent_pattern = sequential-pipeline, failure.failure_modes including
  "hallucinated-citation", "no-relevant-chunks", "citation-not-grounded"; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/rag-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Basic RAG Workflow", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session detail with question header, corpus
  chunk count, answer text with citation chips, guardrail-outcome chip). Browser title
  exactly: <title>Akka Sample: Basic RAG Workflow</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task. Sets model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    ingest-corpus.json — 3 IndexedCorpus entries, one per seeded corpus file, each with 8-15
      IndexedChunk items. Each entry's tool_calls array contains: 1 loadDocuments(corpusId)
      + 1 indexChunks(documents) call.
    answer-query.json — 6 RagAnswer entries. Each carries 1-3 Citation items referencing
      URLs from the paired IndexedCorpus chunks, tool_calls containing 1 retrieveChunks +
      1 buildContext call. Plus 1 deliberately UNGROUNDED entry whose first Citation
      references a URL absent from the indexed corpus — the guardrail blocks it; the mock
      then falls through to a grounded retry. The mock selects the ungrounded entry on the
      FIRST iteration of every 4th session (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. RagAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion RagTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (ingestStep
  60s, queryStep 60s, error 5s).
- Lesson 6: every nullable lifecycle field on the RagSessionRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: RagTasks.java with INGEST_CORPUS and ANSWER_QUERY constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9886 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (RagAgent). Citation
  checking is deterministic (AnswerGuardrail) and does NOT make an LLM call.
- The sequential-pipeline invariant: INGEST task runs and records IndexedCorpus before the
  QUERY task starts. The workflow is the enforcement mechanism — the QUERY task's
  instruction context is built from the recorded IndexedCorpus, so the agent cannot answer
  without it.
- Task dependency is carried by typed task results: ingestStep writes IndexedCorpus onto the
  entity, queryStep reads it and builds the QUERY task's instruction context from it. The
  agent itself is stateless across phases.
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
