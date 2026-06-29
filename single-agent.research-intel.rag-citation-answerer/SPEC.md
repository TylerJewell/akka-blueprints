# SPEC — rag-citation-answerer

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RAG Citation Answerer.
**One-line pitch:** A user uploads documents and asks a question; one AI agent reads the retrieved chunks (passed as a task attachment, never as inline prompt text) and returns a grounded answer with per-citation references back to the source document and chunk.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `CitationAnswererAgent` (AutonomousAgent) produces every answer; the surrounding components prepare its inputs and audit its outputs. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw document upload and chunk indexing — so the model never retrieves or cites raw identifiers.
- A **before-agent-response guardrail** validates every answer the agent produces: every `chunkId` it cites must appear in the retrieved chunk set passed to the task, every citation must carry a non-empty `excerpt`, and the answer must parse into `CitedAnswer`. A citation referencing a chunk that was never retrieved is a hallucinated source; the guardrail catches that before the answer leaves the agent loop.

The blueprint shows that RAG-based question answering is not inherently safe — a model can cite sources it never received. The guardrail enforces grounding at the response boundary so hallucinated citations never reach users.

## 3. User-facing flows

The user opens the App UI tab.

1. The user switches to the **Upload** panel, pastes document text (or loads one of three seeded reports — a synthetic climate-policy brief, a public-health surveillance summary, and a software-procurement guidance note), gives the document a title, and clicks **Upload document**. The card appears in the Documents pane with status `UPLOADED`.
2. Within ~1 s the card transitions to `SANITIZED` — the PII categories found are listed beside the card.
3. Within ~2 s the card transitions to `INDEXED` — the chunk count is shown (e.g., "14 chunks").
4. The user switches to the **Ask** panel. They type a question into the question textarea, select one or more uploaded documents from a checkbox list, and click **Ask**.
5. A query card appears in the Queries pane with status `SUBMITTED`. Within ~1 s it becomes `RETRIEVING`. The retriever selects the top-K chunks from the indexed documents and passes them as a task attachment to the agent.
6. Within ~10–30 s the card transitions to `ANSWERED`. The answer panel on the right shows: the full answer text, a confidence badge (`HIGH` / `MEDIUM` / `LOW`), and a citations table (chunkId, document title, chunk excerpt, page or section hint).
7. The user can hover a citation in the answer text to highlight the matching row in the citations table.
8. The user may submit more questions; the Queries pane keeps the full history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/documents/*` and `/api/queries/*` — upload, ask, list, get, SSE; serves `/api/metadata/*`. | — | `DocumentEntity`, `QueryEntity`, `DocumentView`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `DocumentEntity` | `EventSourcedEntity` | Per-document lifecycle: uploaded → sanitized → chunked → indexed. Source of truth. | `QueryEndpoint`, `DocumentSanitizer`, `ChunkIndexer` | `DocumentView` |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → retrieving → answered → failed. Source of truth. | `QueryEndpoint`, `AnswerWorkflow` | `QueryView` |
| `DocumentSanitizer` | `Consumer` | Subscribes to `DocumentUploaded` events; strips PII; calls `DocumentEntity.attachSanitized`. | `DocumentEntity` events | `DocumentEntity` |
| `ChunkIndexer` | `Consumer` | Subscribes to `DocumentSanitized` events; splits text into chunks; stores keyword-TF vectors in-process; calls `DocumentEntity.markIndexed(chunkCount)`. | `DocumentEntity` events | `DocumentEntity` |
| `AnswerWorkflow` | `Workflow` | One workflow per queryId. Steps: `retrieveStep` → `answerStep`. | started by `QueryEndpoint` once query submitted | `CitationAnswererAgent`, `QueryEntity` |
| `CitationAnswererAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question as task instructions and retrieved chunks as a task attachment; returns `CitedAnswer`. | invoked by `AnswerWorkflow` | returns cited answer |
| `CitationGuardrail` | supporting class | `before-agent-response` hook validating citation grounding on every agent turn. | registered on `CitationAnswererAgent` | agent retry or pass-through |
| `DocumentView` | `View` | Read model: one row per document. | `DocumentEntity` events | `QueryEndpoint` |
| `QueryView` | `View` | Read model: one row per query. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record UploadRequest(
    String documentId,
    String documentTitle,
    String rawText,
    String uploadedBy,
    Instant uploadedAt
) {}

record SanitizedText(
    String redactedText,
    List<String> piiCategoriesFound
) {}

record Chunk(
    String chunkId,
    String documentId,
    String documentTitle,
    int chunkIndex,
    String text,
    String sectionHint      // e.g. "§3", "p.12", "" if not parseable
) {}

record RetrievalResult(
    List<Chunk> chunks,
    int totalChunksSearched
) {}

record Citation(
    String chunkId,
    String documentId,
    String documentTitle,
    String excerpt,         // verbatim passage cited
    String sectionHint
) {}

enum Confidence { HIGH, MEDIUM, LOW }

record CitedAnswer(
    String answerText,
    Confidence confidence,
    List<Citation> citations,
    Instant answeredAt
) {}

record Document(
    String documentId,
    Optional<UploadRequest> upload,
    Optional<SanitizedText> sanitized,
    DocumentStatus status,
    int chunkCount,
    Instant createdAt,
    Optional<Instant> indexedAt
) {}

enum DocumentStatus {
    UPLOADED, SANITIZED, CHUNKED, INDEXED, FAILED
}

record Query(
    String queryId,
    String questionText,
    List<String> documentIds,
    String askedBy,
    Optional<RetrievalResult> retrieval,
    Optional<CitedAnswer> answer,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> answeredAt
) {}

enum QueryStatus {
    SUBMITTED, RETRIEVING, ANSWERED, FAILED
}
```

Events on `DocumentEntity`: `DocumentUploaded`, `DocumentSanitized`, `DocumentChunked`, `DocumentIndexed`, `DocumentFailed`.

Events on `QueryEntity`: `QuerySubmitted`, `RetrievalStarted`, `AnswerRecorded`, `QueryFailed`.

Every nullable lifecycle field on `Document` and `Query` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/documents` — body `{ documentTitle, rawText, uploadedBy }` → `{ documentId }`.
- `GET /api/documents` — list all documents, newest-first.
- `GET /api/documents/{id}` — one document (with sanitized text; raw text omitted from the response body — audit access only via admin path).
- `GET /api/documents/sse` — Server-Sent Events; one event per document state transition.
- `POST /api/queries` — body `{ questionText, documentIds: [String], askedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query row.
- `GET /api/queries/sse` — Server-Sent Events; one event per query state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: RAG Citation Answerer</title>`.

The App UI tab is a two-column layout: a left column with Upload and Ask panels plus document and query history panes, and a right column showing the selected query's full answer with citations table and the selected document's sanitization metadata.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `CitationAnswererAgent`. Asserts (1) the candidate response parses into `CitedAnswer`; (2) every `citations[].chunkId` is present in the chunk set passed as the task attachment; (3) every `citations[].excerpt` is non-empty; (4) the `confidence` value is one of `{HIGH, MEDIUM, LOW}`. On any failure returns a structured `invalid-citation` error to the agent loop so the task retries within its 3-iteration budget. A citation for a chunk the model never received is a hallucinated source; this guardrail is the primary grounding guarantee.
- **S1 — PII sanitizer** (`pii`, applied inside `DocumentSanitizer` Consumer): redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from uploaded document text before any chunk is stored or retrieved. Records which categories were found. The raw text is preserved on the entity for audit but never enters the index.

## 9. Agent prompts

- `CitationAnswererAgent` → `prompts/citation-answerer.md`. The single decision-making LLM. System prompt instructs it to read the attached retrieved chunks and produce a grounded `CitedAnswer` — every claim supported by a citation from the chunks provided; no claim fabricated from training knowledge alone.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User uploads the climate-policy seed, waits for `INDEXED`, asks a question that the document clearly answers; the answer arrives within 30 s with at least one citation whose `chunkId` traces back to the uploaded document.
2. **J2** — The agent's first iteration on a query cites a `chunkId` not in the retrieved set (mock LLM path); the guardrail rejects it; the second iteration produces a valid answer; the UI never displays the invalid citation.
3. **J3** — A document containing `jane.doe@example.com` and `SSN 123-45-6789` is uploaded; a question asking for "contact details" is answered; the answer and all citations contain only redacted forms; the query log never shows the raw PII.
4. **J4** — A question submitted against an empty document list returns a well-formed `CitedAnswer` with `confidence = LOW`, empty `citations`, and `answerText` explaining that no documents were found.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named rag-citation-answerer demonstrating the single-agent × research-intel cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-rag-citation-answerer. Java package io.akka.samples.rag. Akka 3.6.0.
HTTP port 9584.

Components to wire (exactly):

- 1 AutonomousAgent CitationAnswererAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/citation-answerer.md>) and
  .capability(TaskAcceptance.of(QueryTasks.ANSWER_QUESTION).maxIterationsPerTask(3)). The task
  receives the question as its instruction text and the retrieved chunks as a task ATTACHMENT
  named "chunks.json" (TaskDef.attachment("chunks.json", chunksJson.getBytes())) — NOT as inline
  prompt text. Output: CitedAnswer{answerText: String, confidence: Confidence (HIGH/MEDIUM/LOW),
  citations: List<Citation>, answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  3-iteration budget.

- 1 Workflow AnswerWorkflow per queryId with two steps:
  * retrieveStep — calls the in-process ChunkStore.retrieve(queryId, documentIds, questionText,
    topK=5) to produce a RetrievalResult; emits RetrievalStarted on QueryEntity; stores the
    retrieved chunks for the next step. WorkflowSettings.stepTimeout 10s.
  * answerStep — emits RetrievalStarted (if not already), then calls componentClient
    .forAutonomousAgent(CitationAnswererAgent.class, "answerer-" + queryId).runSingleTask(
      TaskDef.instructions("Answer the question: " + query.questionText)
        .attachment("chunks.json", serializeChunks(retrieval.chunks))
    ) — returns a taskId, then forTask(taskId).result(QueryTasks.ANSWER_QUESTION) to fetch the
    answer. On success calls QueryEntity.recordAnswer(answer). WorkflowSettings.stepTimeout 90s
    with defaultStepRecovery maxRetries(2).failoverTo(AnswerWorkflow::error). error step
    transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 2 EventSourcedEntities:
  * DocumentEntity (one per documentId). State Document{documentId: String, upload:
    Optional<UploadRequest>, sanitized: Optional<SanitizedText>, status: DocumentStatus,
    chunkCount: int, createdAt: Instant, indexedAt: Optional<Instant>}. DocumentStatus enum:
    UPLOADED, SANITIZED, CHUNKED, INDEXED, FAILED. Events: DocumentUploaded{upload},
    DocumentSanitized{sanitized}, DocumentChunked{chunkCount}, DocumentIndexed{chunkCount},
    DocumentFailed{reason}. Commands: upload, attachSanitized, markChunked, markIndexed, fail,
    getDocument. emptyState() returns Document.initial("") (Lesson 3).
  * QueryEntity (one per queryId). State Query{queryId: String, questionText: String,
    documentIds: List<String>, askedBy: String, retrieval: Optional<RetrievalResult>,
    answer: Optional<CitedAnswer>, status: QueryStatus, createdAt: Instant,
    answeredAt: Optional<Instant>}. QueryStatus enum: SUBMITTED, RETRIEVING, ANSWERED, FAILED.
    Events: QuerySubmitted{request}, RetrievalStarted{retrievedChunkCount},
    AnswerRecorded{answer}, QueryFailed{reason}. Commands: submit, markRetrieving,
    recordAnswer, fail, getQuery. emptyState() returns Query.initial("") (Lesson 3).

- 2 Consumers:
  * DocumentSanitizer subscribed to DocumentEntity events; on DocumentUploaded runs a
    regex+heuristic redaction pipeline (emails, phone numbers, SSN-like, payment-card-like,
    postal addresses, person-name heuristic, account-id-like tokens) over rawText, builds
    SanitizedText, calls DocumentEntity.attachSanitized(sanitized).
  * ChunkIndexer subscribed to DocumentEntity events; on DocumentSanitized splits the
    redactedText into overlapping 512-character chunks with 64-character overlap, assigns
    chunkIds ("doc-" + documentId + "-" + chunkIndex), computes keyword-TF vectors, stores
    them in an in-process ChunkStore (a singleton ConcurrentHashMap keyed by chunkId),
    calls DocumentEntity.markChunked(chunkCount) then DocumentEntity.markIndexed(chunkCount).

- 2 Views:
  * DocumentView with row type DocumentRow (mirrors Document minus upload.rawText). ONE query:
    getAllDocuments: SELECT * AS documents FROM document_view. No WHERE status filter —
    Akka cannot auto-index enum columns (Lesson 2).
  * QueryView with row type QueryRow (mirrors Query). ONE query: getAllQueries:
    SELECT * AS queries FROM query_view. No WHERE status filter.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with:
    POST /documents (body {documentTitle, rawText, uploadedBy}; mints documentId; calls
      DocumentEntity.upload; returns {documentId}),
    GET /documents (list from getAllDocuments, newest-first),
    GET /documents/{id},
    GET /documents/sse (SSE from DocumentView stream-updates),
    POST /queries (body {questionText, documentIds: [String], askedBy}; mints queryId; calls
      QueryEntity.submit; starts AnswerWorkflow; returns {queryId}),
    GET /queries (list from getAllQueries, newest-first),
    GET /queries/{id},
    GET /queries/sse (SSE from QueryView stream-updates),
    and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer question")
  .description("Read the attached document chunks and produce a CitedAnswer with grounded
  citations").resultConformsTo(CitedAnswer.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- ChunkStore.java — a singleton (application-scoped @Bean) ConcurrentHashMap<String, Chunk>
  storing all indexed chunks. Provides retrieve(queryId, documentIds, questionText, topK):
  filters chunks by documentId membership, scores by keyword overlap with questionText,
  returns top-K as a RetrievalResult.

- Domain records: UploadRequest, SanitizedText, Chunk, RetrievalResult, Citation, Confidence,
  CitedAnswer, Document, DocumentStatus, Query, QueryStatus.

- CitationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  CitedAnswer from the LLM response, runs the four checks listed in eval-matrix.yaml G1, and
  either passes the response through or returns Guardrail.reject(<structured-error>) naming the
  failed check.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9584 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-documents.jsonl with 3 seeded research documents:
  a synthetic climate-policy brief (~1200 words), a synthetic public-health surveillance
  summary (~1000 words), and a synthetic software-procurement guidance note (~800 words).
  Each contains 2–3 PII strings so S1 has work to do. Each document is paired with 2
  example questions whose expected answers are traceable to specific chunks.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = informational
  (the answer is advisory; the reader decides what to do with the cited information),
  oversight.human_in_loop = true, failure.failure_modes including "hallucinated-citation",
  "out-of-scope-answer", "pii-leakage-via-retrieval", "low-confidence-accepted-as-high";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/citation-answerer.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: RAG Citation Answerer", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab: two-column layout (left = Upload
  panel + document list + Ask panel + query list; right = selected query detail with citations
  table and selected document sanitization metadata).
  Browser title exactly: <title>Akka Sample: RAG Citation Answerer</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on Task<R> id. For ANSWER_QUESTION: reads
  src/main/resources/mock-responses/answer-question.json, picks one entry deterministically
  per queryId. Include 6 CitedAnswer entries covering HIGH/MEDIUM/LOW confidence, plus
  2 deliberately MALFORMED entries — one with a chunkId not in any known chunk set, one with
  an empty excerpt — so J2 is reproducible. The mock selects a malformed entry on the first
  iteration of every third query (modulo seed).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md.
Notably:

- Lesson 1: CitationAnswererAgent extends akka.javasdk.agent.autonomous.AutonomousAgent.
  The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (retrieveStep 10s, answerStep 90s,
  error 5s).
- Lesson 6: every nullable lifecycle field on Document and Query is Optional<T>.
- Lesson 7: QueryTasks.java with ANSWER_QUESTION is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9584 declared in akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides AND themeVariables.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (CitationAnswererAgent). ChunkStore
  retrieval and the citation guardrail are NOT LLM calls.
- Retrieved chunks are passed as a Task ATTACHMENT ("chunks.json"), never inlined into the
  instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration block,
  not as a post-hoc external check.
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
