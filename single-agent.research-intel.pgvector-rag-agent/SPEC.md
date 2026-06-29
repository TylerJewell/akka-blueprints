# SPEC — pgvector-rag-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RAG Agent.
**One-line pitch:** A user submits a question; one AI agent retrieves the most relevant passages from a pgvector-indexed documentation corpus and returns a grounded answer where every factual claim cites the chunk it came from.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `CorpusQueryAgent` (AutonomousAgent) carries the entire synthesis decision; the surrounding components prepare the retrieval context and audit the answer's grounding quality. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates the agent's answer on every turn: every factual sentence must reference at least one chunk id from the retrieved passage set; no chunk id outside the retrieved set may be cited; the answer must not be empty. A failing answer triggers a retry inside the same task.

Additionally, an **on-decision evaluator** runs after every `AnswerRecorded` event and scores the answer on citation density, answer length proportionality, and absence of out-of-context chunk references.

The blueprint shows that grounding is a first-class concern enforced at the framework boundary, not a prompt-level suggestion.

## 3. User-facing flows

The user opens the App UI tab.

1. The user optionally adds a document to the corpus via the **Ingest** panel (paste a text document and a source label) or uses one of three seeded corpus entries — an Akka event-sourcing guide (400 words), an HTTP endpoint reference (600 words), and a workflow API reference (500 words).
2. The user types a question in the **Question** field and clicks **Ask**.
3. The UI POSTs to `/api/questions` and receives a `questionId`.
4. A card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `RETRIEVING` as the workflow embeds the question and queries pgvector for the top-K relevant passages.
5. The card transitions to `ANSWERING` when `CorpusQueryAgent` is invoked with the question text as task instructions and the retrieved passages as a task attachment.
6. Within ~10–30 s the card transitions to `ANSWERED`. The right pane shows the answer text with inline citation markers, a passage table (chunk id, source label, relevance score, passage text), and a citation map linking each claim to its chunk.
7. Within ~1 s of the answer, the `evalStep` finishes. The card shows an **eval score** chip (1–5) and a one-line rationale.
8. The user can ask another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/questions/*` and `/api/corpus/*` — ask, list, get, SSE, ingest; serves `/api/metadata/*`. | — | `QuestionEntity`, `CorpusEntity`, `QuestionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QuestionEntity` | `EventSourcedEntity` | Per-question lifecycle: submitted → retrieving → answering → answered → evaluated. Source of truth. | `QueryEndpoint`, `QueryWorkflow` | `QuestionView` |
| `CorpusEntity` | `EventSourcedEntity` | Per-document corpus record: added → indexed. Tracks chunking and embedding status. | `QueryEndpoint` | `CorpusIngestionConsumer` |
| `CorpusIngestionConsumer` | `Consumer` | Subscribes to `DocumentAdded` events on `CorpusEntity`; splits document into chunks; calls embedding service; writes vectors to pgvector; calls `CorpusEntity.markIndexed`. | `CorpusEntity` events | `CorpusEntity`, pgvector |
| `QueryWorkflow` | `Workflow` | One workflow per question. Steps: `embedStep` → `retrieveStep` → `answerStep` → `evalStep`. | started by `QueryEndpoint` after `QuestionEntity.submit` | `CorpusQueryAgent`, `QuestionEntity` |
| `CorpusQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question in task instructions and the retrieved passages as a task attachment; returns `Answer`. | invoked by `QueryWorkflow` | returns `Answer` |
| `CitationGuardrail` | guardrail hook | `before-agent-response` hook. Validates that every factual sentence cites at least one retrieved chunk id; rejects uncited answers. | registered on `CorpusQueryAgent` | — |
| `AnswerEvaluationScorer` | helper class | Deterministic rule-based scorer (no LLM). Scores citation density, answer length, and out-of-context chunk absence. | invoked by `QueryWorkflow.evalStep` | returns `EvalResult` |
| `QuestionView` | `View` | Read model: one row per question for the UI. | `QuestionEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CorpusDocument(
    String documentId,
    String sourceLabel,
    String body,
    Instant addedAt
) {}

record Chunk(
    String chunkId,        // documentId + "-" + index
    String documentId,
    String sourceLabel,
    String text,
    int tokenCount
) {}

record RetrievedPassage(
    String chunkId,
    String sourceLabel,
    String text,
    float relevanceScore
) {}

record Citation(
    String chunkId,
    String sourceLabel
) {}

record Answer(
    String answerText,
    List<Citation> citations,
    List<RetrievedPassage> passages,   // the full retrieved set for transparency
    Instant answeredAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Question(
    String questionId,
    String questionText,
    String askedBy,
    Optional<Answer> answer,
    Optional<EvalResult> eval,
    QuestionStatus status,
    Instant submittedAt,
    Optional<Instant> finishedAt
) {}

enum QuestionStatus {
    SUBMITTED, RETRIEVING, ANSWERING, ANSWERED, EVALUATED, FAILED
}

enum CorpusStatus {
    ADDED, INDEXED, FAILED
}
```

Events on `QuestionEntity`: `QuestionSubmitted`, `RetrievalStarted`, `AnsweringStarted`, `AnswerRecorded`, `EvaluationScored`, `QuestionFailed`.
Events on `CorpusEntity`: `DocumentAdded`, `DocumentIndexed`, `IndexingFailed`.

Every nullable lifecycle field on the `Question` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/questions` — body `{ questionText, askedBy }` → `{ questionId }`.
- `GET /api/questions` — list all questions, newest-first.
- `GET /api/questions/{id}` — one question.
- `GET /api/questions/sse` — Server-Sent Events; one event per state transition.
- `POST /api/corpus` — body `{ sourceLabel, body }` → `{ documentId }`.
- `GET /api/corpus` — list indexed documents.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: RAG Agent</title>`.

The App UI tab is a two-column layout: a left rail with a question submission form and the live list of submitted questions (status pill + eval score chip + age) and a right pane with the selected question's detail — retrieved passages table, answer text with citation markers, and eval score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `CorpusQueryAgent`. Asserts the candidate `Answer` carries at least one `Citation`; every `citation.chunkId` is present in the set of retrieved passage chunk ids passed to the task; and the `answerText` is non-empty. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `AnswerRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (no LLM call) measures citation density (citations per 100 words), whether `answerText` is proportionate to the number of retrieved passages, and that no citation references a chunk id outside the retrieved set. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `CorpusQueryAgent` → `prompts/corpus-query-agent.md`. The single decision-making LLM. System prompt instructs it to read the retrieved passages in the attachment, synthesise an answer that directly addresses the question, and cite every factual claim with its source chunk id.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks a question about content present in the seeded corpus; within 30 s the answer appears with at least one citation per factual claim and an eval score chip.
2. **J2** — The agent's first response on a question is intentionally uncited (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a cited answer; the UI never displays the uncited response.
3. **J3** — A question about content absent from the corpus produces an explicit "insufficient context" answer (score ≤ 2 with rationale "No supporting passages found"); the card's border highlights red.
4. **J4** — After ingesting a new document, the next question whose answer depends on that document returns citations from the newly indexed chunks within the same session.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named pgvector-rag-agent demonstrating the single-agent × research-intel cell.
Requires Docker Desktop or Docker Engine + Compose v2 for the pgvector container (started
automatically by the Akka dev-mode docker-compose integration — add a docker-compose.yml in the
project root that starts pgvector/pgvector:pg16 on port 5432 with POSTGRES_PASSWORD=secret and
POSTGRES_DB=ragagent). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-pgvector-rag-agent. Java package io.akka.samples.ragagent.
Akka 3.6.0. HTTP port 9197.

Components to wire (exactly):

- 1 AutonomousAgent CorpusQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/corpus-query-agent.md>) and
  .capability(TaskAcceptance.of(QueryTasks.ANSWER_QUESTION).maxIterationsPerTask(3)).
  The task receives the question as its instruction text and the retrieved passages as
  a task ATTACHMENT named "passages.txt" (a formatted list of chunk id, source label,
  relevance score, and text — NOT as inline instructions). Output: Answer{answerText:
  String, citations: List<Citation>, passages: List<RetrievedPassage>, answeredAt: Instant}.
  The agent is configured with a before-agent-response guardrail (see G1 in
  eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow QueryWorkflow per questionId with four steps:
  * embedStep — calls an EmbeddingService helper (wraps the configured model provider's
    embedding endpoint or a MockEmbeddingService in mock mode) to get a float[] for the
    question text. Stores the embedding in the workflow state. WorkflowSettings.stepTimeout 10s.
  * retrieveStep — queries pgvector for the top-5 cosine-nearest chunks to the question
    embedding using a PgVectorStore helper (wraps a JDBC DataSource backed by application.conf
    postgres config). Emits RetrievalStarted on QuestionEntity. WorkflowSettings.stepTimeout 15s.
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    CorpusQueryAgent.class, "querier-" + questionId).runSingleTask(
      TaskDef.instructions("Question: " + question.questionText)
        .attachment("passages.txt", formatPassages(passages).getBytes())
    ) — returns a taskId, then forTask(taskId).result(QueryTasks.ANSWER_QUESTION) to fetch
    the answer. On success calls QuestionEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * evalStep — runs a deterministic rule-based AnswerEvaluationScorer (NOT an LLM call)
    over the recorded answer: checks citation density (citations / 100 words ≥ 0.5 = full
    score), checks that every Citation.chunkId is in the retrieved set, checks that
    answerText is not empty. Emits EvaluationScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QuestionEntity (one per questionId). State Question{questionId: String,
  questionText: String, askedBy: String, answer: Optional<Answer>, eval: Optional<EvalResult>,
  status: QuestionStatus, submittedAt: Instant, finishedAt: Optional<Instant>}.
  QuestionStatus enum: SUBMITTED, RETRIEVING, ANSWERING, ANSWERED, EVALUATED, FAILED.
  Events: QuestionSubmitted{questionText, askedBy}, RetrievalStarted{}, AnsweringStarted{},
  AnswerRecorded{answer}, EvaluationScored{eval}, QuestionFailed{reason}.
  Commands: submit, markRetrieving, markAnswering, recordAnswer, recordEvaluation, fail,
  getQuestion. emptyState() returns Question.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 EventSourcedEntity CorpusEntity (one per documentId). State CorpusDocument{documentId:
  String, sourceLabel: String, body: String, status: CorpusStatus, addedAt: Instant,
  chunkCount: Optional<Integer>}. CorpusStatus enum: ADDED, INDEXED, FAILED.
  Events: DocumentAdded{sourceLabel, body}, DocumentIndexed{chunkCount}, IndexingFailed{reason}.
  Commands: add, markIndexed, failIndexing, getDocument.

- 1 Consumer CorpusIngestionConsumer subscribed to CorpusEntity events; on DocumentAdded
  runs a ChunkSplitter helper (fixed 150-token overlapping chunks with 20-token overlap),
  calls EmbeddingService.embed(text) for each chunk to get a float[], stores each chunk +
  vector in the pgvector "corpus_chunks" table (columns: chunk_id, document_id, source_label,
  text, embedding vector(1536)), then calls CorpusEntity.markIndexed(chunkCount). The
  EmbeddingService reads the same model-provider env vars as the agent; in mock mode it
  returns a deterministic pseudo-random float[] seeded by the chunk text hash.

- 1 View QuestionView with row type QuestionRow (mirrors Question minus any large text fields
  held only on the entity). Table updater consumes QuestionEntity events. ONE query
  getAllQuestions: SELECT * AS questions FROM question_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with:
    POST /questions (body {questionText, askedBy}; mints questionId; calls
      QuestionEntity.submit then starts QueryWorkflow; returns {questionId}),
    GET /questions (list from getAllQuestions, sorted newest-first),
    GET /questions/{id} (one row),
    GET /questions/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /corpus (body {sourceLabel, body}; mints documentId; calls CorpusEntity.add;
      returns {documentId}),
    GET /corpus (list indexed documents),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question").description("Read the retrieved passages and produce a grounded Answer citing
  chunk ids").resultConformsTo(Answer.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records Chunk, RetrievedPassage, Citation, Answer, EvalResult, Question,
  QuestionStatus, CorpusDocument, CorpusStatus.

- CitationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  Answer from the LLM response, runs the three checks listed in eval-matrix.yaml G1
  (non-empty answerText, at least one citation, all citation chunkIds in the retrieved set),
  and either passes the response through or returns Guardrail.reject(<structured-error>)
  to force the agent loop to retry.

- AnswerEvaluationScorer.java — pure deterministic logic (no LLM). Inputs: Answer and the
  List<RetrievedPassage> that the workflow retrieved. Outputs: EvalResult. Scoring rubric
  documented in Javadoc on the class.

- EmbeddingService.java — interface + two implementations: ModelProviderEmbeddingService
  (calls the configured provider's text-embedding endpoint) and MockEmbeddingService (returns
  a deterministic float[] seeded by FNV-1a hash of the input text). The mock returns
  vector(1536) to match OpenAI text-embedding-3-small dimensions.

- PgVectorStore.java — thin JDBC wrapper around the "corpus_chunks" table. Provides
  store(Chunk, float[]) and queryNearest(float[] questionEmbedding, int topK) →
  List<RetrievedPassage>. Uses SELECT ... ORDER BY embedding <=> ? LIMIT k with the
  pgvector cosine-distance operator. Connects via DataSource configured from
  application.conf postgres block.

- ChunkSplitter.java — splits a body String into overlapping 150-token chunks with 20-token
  overlap, using a simple whitespace tokenizer. Returns List<Chunk>.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9197, the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}, and a postgres block:
  ragagent.postgres.url = "jdbc:postgresql://localhost:5432/ragagent"
  ragagent.postgres.password = ${?PGPASSWORD} (default "secret" for dev-mode).

- docker-compose.yml at the project root:
  services:
    pgvector:
      image: pgvector/pgvector:pg16
      ports: ["5432:5432"]
      environment:
        POSTGRES_PASSWORD: secret
        POSTGRES_DB: ragagent
      volumes: ["pgvector-data:/var/lib/postgresql/data"]
  volumes:
    pgvector-data:

- src/main/resources/sample-events/corpus-documents.jsonl with 3 seeded documents:
  an Akka event-sourcing guide (~400 words), an Akka HTTP endpoint reference (~600 words),
  and an Akka workflow API reference (~500 words). Each document has a distinct sourceLabel.

- src/main/resources/sample-events/seed-questions.jsonl with 5 sample questions that are
  directly answerable from the seeded corpus documents, plus 2 questions about topics not
  covered (to exercise the "insufficient context" path).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, E1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes appropriate for research-intel
  (public-documentation corpus, no PII), decisions.authority_level = informational
  (the agent's answer is a research aid, not an authoritative ruling),
  oversight.human_in_loop = true (a researcher reads the answer before acting),
  failure.failure_modes including "hallucinated-citation", "out-of-context-answer",
  "missed-passage", "citation-mismatch"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/corpus-query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: RAG Agent", prerequisites (including
  Docker requirement), generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  question submission form + live list of question cards; right = selected-question detail
  with retrieved passages table, answer text with citation markers, and eval-score chip).
  Browser title exactly: <title>Akka Sample: RAG Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock. The mock embedding service
        is automatically used alongside.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(questionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 8 Answer entries covering both well-cited and poorly-cited
      responses. Each well-cited entry has an answerText of 3-5 sentences with inline
      [chunkId] markers and a citations list matching passages from the seeded corpus.
      Plus 2 deliberately UNCITED entries (answers with empty citations list) — the
      CitationGuardrail blocks both, exercising the retry path. The mock should select
      an uncited entry on the FIRST iteration of every 3rd question (modulo seed) so J2
      is reproducible.
    Plus 1 "insufficient context" entry: answerText = "The retrieved passages do not
      contain sufficient information to answer this question." with an empty citations list
      that the guardrail passes (the guardrail only checks well-cited answers; the
      insufficient-context phrase is a sentinel that passes through as a special case —
      document this in CitationGuardrail's Javadoc).
- A MockModelProvider.seedFor(questionId) helper makes per-question selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CorpusQueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (embedStep 10s, retrieveStep 15s,
  answerStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Question row record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: QueryTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(Answer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9197 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Requires Docker" — never T1/T2/T3/T4 in any user-visible
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (CorpusQueryAgent). The
  on-decision eval is rule-based (AnswerEvaluationScorer.java) and does NOT make an LLM call.
- The retrieved passages are passed as a Task ATTACHMENT, never inlined into the agent's
  instructions text. Verify the generated answerStep uses TaskDef.attachment(...) and not
  string interpolation.
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
4. Run `/akka:build` — compile and start the service via the Akka MCP tools. Ensure Docker is running and the pgvector container starts before the JVM attempts to connect.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), Docker not running (instruct the user to start Docker Desktop or Docker Engine), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
