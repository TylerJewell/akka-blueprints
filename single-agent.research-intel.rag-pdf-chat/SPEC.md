# SPEC — rag-pdf-chat

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RagPdfChat.
**One-line pitch:** A user uploads a PDF and asks questions about it; one AI agent retrieves the most relevant passages via in-process vector similarity and returns an answer that cites each source passage by id, so every claim is traceable back to the document.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `PdfChatAgent` (AutonomousAgent) carries the entire answer; the surrounding components index the document, retrieve relevant passages, and audit the agent's citation behaviour. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates the agent's answer on every turn: every `CitedPassage.passageId` in the answer must exist in the set of passages supplied to the task, the answer must not be empty, and the `answerable` flag must be consistent with the citations list (if `answerable` is `false`, `citations` must be empty; if `true`, at least one citation must be present). A non-conforming answer triggers a retry inside the same task.

The blueprint shows that a RAG agent can make its citation behaviour machine-verifiable — not just a prompt instruction — by wiring the guardrail at the framework level. The retriever and indexer are deterministic components that handle no LLM calls; only the single `PdfChatAgent` talks to a model.

## 3. User-facing flows

The user opens the App UI tab.

1. The user clicks **Upload PDF** and picks a local file (or clicks **Load seeded PDF** to use one of three seeded documents: a technical white paper on distributed systems, a policy brief on data governance, and a public-domain research article on retrieval-augmented generation).
2. The upload POSTs to `/api/documents` and returns a `documentId`. The document card appears in the left panel with status `UPLOADING`. Within ~1 s it transitions to `INDEXED` — the passage count chip shows how many chunks were extracted.
3. The user selects the indexed document from the document list, types a question into the chat input, and clicks **Ask**.
4. The question POSTs to `/api/documents/{id}/chat`. A question card appears in `RETRIEVING` state. Within ~1 s passages are ranked; the card transitions to `ANSWERING`.
5. Within ~10–30 s the agent finishes. The card transitions to `ANSWERED`. The answer text appears with inline citation markers (`[P-001]`, `[P-007]`, etc.). Clicking a marker scrolls to the matching passage excerpt in the Citations panel on the right.
6. If the document does not contain relevant passages, the card shows an `UNANSWERABLE` badge and the message "I cannot find this in the document."
7. The user can ask follow-up questions in the same session; prior exchanges appear above in the chat history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/documents/*` and `/api/documents/{id}/chat` — upload, list, ask, SSE; serves `/api/metadata/*`. | — | `PdfDocumentEntity`, `ChatSessionWorkflow`, `ChatSessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PdfDocumentEntity` | `EventSourcedEntity` | Per-document lifecycle: uploaded → indexed. Holds passage list and document metadata. Source of truth. | `ChatEndpoint`, `PassageRetriever` | `ChatSessionView` |
| `PassageRetriever` | `Consumer` | Subscribes to `DocumentUploaded` events; chunks the PDF into fixed-size passages, computes TF-IDF-style term vectors for each passage, writes `DocumentIndexed` back to the entity. At query time the workflow calls `PassageRetriever.rankPassages(documentId, question, topK)` to return ranked passages. | `PdfDocumentEntity` events | `PdfDocumentEntity` |
| `ChatSessionWorkflow` | `Workflow` | One workflow per question. Steps: `retrieveStep` → `answerStep`. Started by `ChatEndpoint` on each question. | started by `ChatEndpoint` | `PdfChatAgent`, `PdfDocumentEntity`, `ChatSessionView` |
| `PdfChatAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user question as the task's instruction text and the ranked passages as a task attachment; returns `CitedAnswer`. | invoked by `ChatSessionWorkflow` | returns answer |
| `CitationGuardrail` | supporting class | `before-agent-response` hook registered on `PdfChatAgent`. Validates every `passageId` in the answer against the supplied passage set. | `PdfChatAgent` response | accept / reject |
| `ChatSessionView` | `View` | Read model: one row per question for the UI. | `PdfDocumentEntity` events, `ChatSessionWorkflow` outputs | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PdfPassage(
    String passageId,       // e.g. "P-001"
    int pageNumber,
    int chunkIndex,
    String text,
    double[] termVector     // in-process similarity; not persisted to entity state
) {}

record DocumentMetadata(
    String documentId,
    String filename,
    int totalPages,
    int passageCount,
    Instant uploadedAt
) {}

record Question(
    String questionId,
    String documentId,
    String sessionId,
    String questionText,
    Instant askedAt
) {}

record CitedPassage(
    String passageId,       // MUST reference a passage id supplied in the task attachment
    String excerpt,         // verbatim short excerpt from the passage
    int pageNumber
) {}

record CitedAnswer(
    boolean answerable,
    String answerText,      // empty string if answerable == false
    List<CitedPassage> citations,   // empty list if answerable == false
    Instant answeredAt
) {}

record ChatExchange(
    String questionId,
    Question question,
    Optional<List<PdfPassage>> retrievedPassages,
    Optional<CitedAnswer> answer,
    ExchangeStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ExchangeStatus {
    RETRIEVING, ANSWERING, ANSWERED, UNANSWERABLE, FAILED
}

record PdfDocument(
    String documentId,
    Optional<DocumentMetadata> metadata,
    Optional<List<PdfPassage>> passages,
    DocumentStatus status,
    Instant createdAt
) {}

enum DocumentStatus {
    UPLOADING, INDEXED, FAILED
}
```

Events on `PdfDocumentEntity`: `DocumentUploaded`, `DocumentIndexed`, `DocumentIndexFailed`.

Events on `ChatSessionWorkflow` → written to `ChatSessionView` via a dedicated view consumer: `QuestionAsked`, `PassagesRetrieved`, `AnswerRecorded`, `ExchangeFailed`.

Every nullable lifecycle field on `PdfDocument` and `ChatExchange` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/documents` — body `{ filename, pdfBase64, sessionId }` → `{ documentId }`.
- `GET /api/documents` — list all documents, newest-first.
- `GET /api/documents/{id}` — one document with passage list.
- `POST /api/documents/{id}/chat` — body `{ questionText, sessionId }` → `{ questionId }`.
- `GET /api/documents/{id}/chat` — list all exchanges for a document (newest-first).
- `GET /api/documents/{id}/chat/sse` — Server-Sent Events; one event per exchange state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Chat With PDF Docs Using AI (Quoting Sources)</title>`.

The App UI tab is a two-column layout: a left column with the document upload panel, the document list, and the chat input / exchange history; a right column with the Citations panel showing the ranked passages for the selected exchange.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `PdfChatAgent`. Asserts: (a) every `CitedPassage.passageId` in `citations` matches a passage id in the set supplied to the task attachment; (b) if `answerable == true`, `citations` is non-empty; (c) if `answerable == false`, `citations` is empty and `answerText` is the literal phrase "I cannot find this in the document"; (d) the response parses into `CitedAnswer` without error. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `PdfChatAgent` → `prompts/pdf-chat-agent.md`. The single decision-making LLM. System prompt instructs it to read the supplied passages, answer the question, and cite every passage it draws on by passage id.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User uploads the seeded white-paper PDF; within 5 s the document is `INDEXED`; a question produces an `ANSWERED` exchange with at least one `CitedPassage` referencing a real passage id from the index.
2. **J2** — Agent returns an answer citing a non-existent passage id on first iteration (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a valid answer; the UI shows only the valid answer.
3. **J3** — A question that has no answer in the document returns `UNANSWERABLE`; `answerable == false`; `citations` is empty; the UI shows an "UNANSWERABLE" badge.
4. **J4** — Two questions against the same document re-use the already-indexed passages without a second indexing round; the document status remains `INDEXED` throughout.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named rag-pdf-chat demonstrating the single-agent × research-intel cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-rag-pdf-chat. Java package
io.akka.samples.chatwithpdfdocsusingaiquotingsources. Akka 3.6.0. HTTP port 9192.

Components to wire (exactly):

- 1 AutonomousAgent PdfChatAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/pdf-chat-agent.md>) and
  .capability(TaskAcceptance.of(PDF_CHAT_TASKS.ANSWER_QUESTION).maxIterationsPerTask(3)). The
  task receives the user question as the task's instruction text and the ranked passages as a
  task ATTACHMENT named "passages.json" (JSON array of PdfPassage objects — NOT inlined into
  the instruction text). Output: CitedAnswer{answerable: boolean, answerText: String,
  citations: List<CitedPassage>, answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow ChatSessionWorkflow per questionId with two steps:
  * retrieveStep — calls PassageRetriever.rankPassages(documentId, questionText, topK=5) via
    componentClient; on success advances to answerStep. WorkflowSettings.stepTimeout 10s.
  * answerStep — calls componentClient.forAutonomousAgent(PdfChatAgent.class,
    "chat-" + questionId).runSingleTask(
      TaskDef.instructions("Question: " + question.questionText())
        .attachment("passages.json", serializePassages(passages).getBytes())
    ) — returns a taskId, then forTask(taskId).result(PDF_CHAT_TASKS.ANSWER_QUESTION) to fetch
    the CitedAnswer. On success writes the answer to ChatSessionView via a AnswerRecorded
    event. WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ChatSessionWorkflow::error). error step transitions the exchange to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PdfDocumentEntity (one per documentId). State PdfDocument{documentId:
  String, metadata: Optional<DocumentMetadata>, passages: Optional<List<PdfPassage>>,
  status: DocumentStatus, createdAt: Instant}. DocumentStatus enum: UPLOADING, INDEXED, FAILED.
  Events: DocumentUploaded{filename, pdfBase64}, DocumentIndexed{metadata, passages},
  DocumentIndexFailed{reason}. Commands: upload, attachIndex, failIndex, getDocument.
  emptyState() returns PdfDocument.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer PassageRetriever subscribed to PdfDocumentEntity events; on DocumentUploaded
  decodes the base64 PDF, extracts raw text (regex paragraph split on line breaks and §/clause
  markers), chunks into passages of ~300 tokens (simple word-count split), assigns passageIds
  ("P-" + zero-padded index), records pageNumber via a running page-break heuristic (form-feed
  or "Page N" header), computes a TF-IDF term frequency map per passage (stored in an
  in-process ConcurrentHashMap keyed by documentId), then calls PdfDocumentEntity.attachIndex.
  At query time exposes a direct Java method rankPassages(documentId, questionText, topK) that
  tokenises the question, computes cosine similarity against stored term vectors, and returns
  the top-K passages sorted by score descending. The PdfPassage.termVector double[] field is
  populated in-memory only and not written to the entity (it is excluded from serialisation).

- 1 View ChatSessionView with row type ChatExchangeRow (mirrors ChatExchange minus the full
  passage text — the right pane fetches passages via GET /api/documents/{id}). Table updater
  consumes ChatSessionWorkflow events written as action results. ONE query:
  getAllExchangesForDocument(documentId): SELECT * AS exchanges FROM chat_session_view WHERE
  document_id = :documentId. Ordered by createdAt DESC. No WHERE status filter on status
  columns (Lesson 2).

- 2 HttpEndpoints:
  * ChatEndpoint at /api with:
    POST /documents (body {filename, pdfBase64, sessionId}; mints documentId; calls
      PdfDocumentEntity.upload; returns {documentId}),
    GET /documents (list, newest-first),
    GET /documents/{id} (one document with passages),
    POST /documents/{id}/chat (body {questionText, sessionId}; mints questionId; starts
      ChatSessionWorkflow; returns {questionId}),
    GET /documents/{id}/chat (list all exchanges),
    GET /documents/{id}/chat/sse (Server-Sent Events for that document's exchanges),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- PdfChatTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question from PDF").description("Read the supplied passages and produce a CitedAnswer for
  the user question").resultConformsTo(CitedAnswer.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PdfPassage, DocumentMetadata, Question, CitedPassage, CitedAnswer,
  ChatExchange, ExchangeStatus, PdfDocument, DocumentStatus.

- CitationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  CitedAnswer from the LLM response, runs the four checks listed in eval-matrix.yaml G1, and
  either passes the response through or returns Guardrail.reject(<structured-error>) naming
  which check failed, to force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9192 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. PdfChatAgent.definition() binds the
  configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-pdfs/ with 3 seeded PDFs represented as base64-encoded stub
  text files:
    distributed-systems-whitepaper.txt — 1400-word synthetic technical overview of
      consensus algorithms, with 2 embedded "Page N" markers and 1 email address.
    data-governance-policy-brief.txt — 1200-word synthetic policy brief on enterprise data
      classification, with department-specific identifiers.
    rag-research-article.txt — 900-word synthetic research article on retrieval-augmented
      generation, referencing several fictional paper titles.

- src/main/resources/sample-events/seed-questions.jsonl with 6 seeded question/document
  pairs (2 per PDF) for fast demo use.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC.

- risk-survey.yaml at the project root with appropriate pre-fills for the research-intel
  domain.

- prompts/pdf-chat-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Chat With PDF Docs Using AI (Quoting
  Sources)", prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs. App UI tab uses a two-column layout (left = document upload panel +
  document list + chat input + exchange history; right = Citations panel showing ranked
  passages for the selected exchange, with page number chips and passage text excerpts).
  Browser title exactly:
  <title>Akka Sample: Chat With PDF Docs Using AI (Quoting Sources)</title>.
  No subtitle on the Overview tab.

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
  dispatch on the Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    answer-question-from-pdf.json — 6 CitedAnswer entries:
      4 answerable=true entries each with 1-3 CitedPassage references using real passageIds
        ("P-001" through "P-008") and short excerpt strings; answerText is 1-2 sentences.
      1 answerable=false entry with empty citations and answerText "I cannot find this in
        the document".
      1 MALFORMED entry with a CitedPassage whose passageId is "P-999" (does not exist in
        any seeded document's passage set) — guardrail blocks this, exercising the retry path.
        The mock should select the malformed entry on the FIRST iteration of every 3rd
        question (modulo seed) so J2 is reproducible.
- MockModelProvider.seedFor(questionId) makes per-question selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PdfChatAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PdfChatTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (retrieveStep 10s, answerStep
  60s, error 5s).
- Lesson 6: every nullable lifecycle field on PdfDocument and ChatExchange is Optional<T>.
- Lesson 7: PdfChatTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(CitedAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9192 declared explicitly in application.conf's
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
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (PdfChatAgent). The
  PassageRetriever is a deterministic Consumer and direct-call component — it makes no LLM
  call. This preserves the pattern's "one agent" promise.
- Passages are passed as a Task ATTACHMENT, never inlined into the agent's instruction text.
  Verify the generated answerStep uses TaskDef.attachment(...) and not string interpolation.
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
