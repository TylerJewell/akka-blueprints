# SPEC — code-search-rag-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CodeSearch Demo Agent.
**One-line pitch:** A user asks a natural-language question about a code repository; one AI agent reads the retrieved code chunks (passed as task attachments, never inlined into prompt text) and returns a grounded answer with specific file paths and line numbers.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `CodeSearchAgent` (AutonomousAgent) carries the entire retrieval-and-answer decision; the surrounding components maintain the chunk index, prepare its input, and audit its output. One governance mechanism is wired around the agent:

- A **secret sanitizer** runs inside a Consumer between the raw chunk retrieval and the agent call — so the model never sees hardcoded credentials, API keys, tokens, or certificate material that may appear in source files.

The blueprint shows that a RAG pattern over code is still a single-agent system: one AutonomousAgent talks to the model, the surrounding Akka primitives handle routing, state, and the safety preprocessing step.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a natural-language question into the **Question** textarea (e.g., "Where is the database connection pool configured?" or "Which files handle JWT validation?").
2. The user optionally selects a **Repository filter** dropdown (All, or one of three seeded corpus tags: `akka-http`, `akka-streams`, `akka-persistence`).
3. The user clicks **Search**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The card appears in the live list with status `SUBMITTED`. Within ~1 s it transitions to `CHUNKS_SANITIZED` — the number of retrieved chunks is shown, plus a small list of secret categories the sanitizer found (if any).
5. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWERED`. The answer appears: a prose paragraph plus a per-reference table (file path, start line, end line, language, relevance blurb).
6. Within ~1 s of the answer, the `groundingStep` finishes. The card shows a **grounding score** chip (1–5) plus a one-line rationale describing whether every cited file path actually appeared in the retrieved chunks.
7. The user can submit another question; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `ChunkIndexView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → chunks_sanitized → answering → answered → grounded. Source of truth. | `QueryEndpoint`, `ChunkSanitizer`, `QueryWorkflow` | `ChunkIndexView` |
| `ChunkSanitizer` | `Consumer` | Subscribes to `QuerySubmitted` events; redacts secrets from retrieved chunks; calls `QueryEntity.attachSanitizedChunks`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitSanitizedStep` → `answerStep` → `groundingStep`. | started by `ChunkSanitizer` once sanitized event lands | `CodeSearchAgent`, `QueryEntity` |
| `CodeSearchAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question as the task instruction and sanitized code chunks as task attachments; returns `SearchAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `ChunkIndexView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CodeChunk(
    String chunkId,
    String filePath,
    int startLine,
    int endLine,
    String language,
    String content,
    String corpusTag
) {}

record QueryRequest(
    String queryId,
    String questionText,
    String corpusTag,           // filter tag or "all"
    List<CodeChunk> retrievedChunks,
    String submittedBy,
    Instant submittedAt
) {}

record SanitizedChunks(
    List<CodeChunk> redactedChunks,   // content has secrets replaced
    List<String> secretCategoriesFound
) {}

record CodeReference(
    String filePath,
    int startLine,
    int endLine,
    String language,
    String relevanceBlurb
) {}

record SearchAnswer(
    String answerText,
    List<CodeReference> references,
    Instant answeredAt
) {}

record GroundingResult(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<SanitizedChunks> sanitized,
    Optional<SearchAnswer> answer,
    Optional<GroundingResult> grounding,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, CHUNKS_SANITIZED, ANSWERING, ANSWERED, GROUNDED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `ChunksSanitized`, `AnsweringStarted`, `AnswerRecorded`, `GroundingScored`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ questionText, corpusTag, retrievedChunks: [CodeChunk], submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: CodeSearch Demo Agent</title>`.

The App UI tab is a two-column layout: a left rail with the question submission form plus the live list of submitted queries (status pill + grounding score chip + age) and a right pane with the selected query's detail — question text, sanitized chunk list, answer paragraph, reference table, and grounding-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Secret sanitizer** (`secret`, applied inside `ChunkSanitizer` Consumer): redacts hardcoded API keys, authentication tokens, private-key PEM blocks, connection strings with embedded passwords, and certificate fingerprints from retrieved code chunks before any LLM call. Records which categories were found. The raw chunks are preserved on the entity for audit but are never the input seen by the model.

## 9. Agent prompts

- `CodeSearchAgent` → `prompts/code-search-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached code chunks, identify the most relevant passages, and return a `SearchAnswer` with a prose explanation and one `CodeReference` per distinct file cited.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "Where is connection pool size configured?" against the `akka-http` corpus tag; within 30 s the answer appears with at least one reference entry pointing to a real file path from the retrieved chunks.
2. **J2** — A retrieved chunk contains the string `apiKey = "sk-live-abc123"` — the LLM call log shows only `[REDACTED-API-KEY]`; the entity's raw chunk retains the original for audit.
3. **J3** — The agent returns an answer citing a file path that was not in any of the attached chunks; the grounding score is 1; the UI flags the card border red.
4. **J4** — No chunks are retrieved for a question (empty corpus tag match); the agent returns a well-formed "not found" `SearchAnswer` with an empty `references` list; grounding score is 5 (no false citations).

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named code-search-rag-agent demonstrating the single-agent × dev-code cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-code-search-rag-agent. Java package
io.akka.samples.codesearchdemoagent. Akka 3.6.0. HTTP port 9276.

Components to wire (exactly):

- 1 AutonomousAgent CodeSearchAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/code-search-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_CODE_QUESTION).maxIterationsPerTask(3)).
  The task receives the question text as its instruction text and each sanitized code
  chunk as a separate task ATTACHMENT (NOT as inline prompt text — one attachment per
  chunk, named "<chunkId>.txt"). Output: SearchAnswer{answerText: String,
  references: List<CodeReference>, answeredAt: Instant}. There is no guardrail on this
  agent — the single governance mechanism is the upstream secret sanitizer.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitSanitizedStep — polls QueryEntity.getQuery every 1s; on
    query.sanitized().isPresent() advances to answerStep. WorkflowSettings.stepTimeout
    15s (sanitizer is in-process and fast).
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    CodeSearchAgent.class, "searcher-" + queryId).runSingleTask(
      TaskDef.instructions(query.request().questionText())
        .attachment("<chunkId>.txt", chunk.content().getBytes())   // one per chunk
    ) — returns a taskId, then forTask(taskId).result(ANSWER_CODE_QUESTION) to fetch
    the answer. On success calls QueryEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * groundingStep — runs a deterministic rule-based GroundingScorer (NOT an LLM call)
    over the recorded answer: checks that every reference.filePath appears in at least
    one of the sanitized chunks' filePath values, that answerText is non-empty, and
    that the references list is non-empty when chunks were available. Awards 5 when all
    file paths are grounded; deducts 1 per ungrounded citation; floors at 1.
    Emits GroundingScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, sanitized: Optional<SanitizedChunks>,
  answer: Optional<SearchAnswer>, grounding: Optional<GroundingResult>,
  status: QueryStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  QueryStatus enum: SUBMITTED, CHUNKS_SANITIZED, ANSWERING, ANSWERED, GROUNDED, FAILED.
  Events: QuerySubmitted{request}, ChunksSanitized{sanitized}, AnsweringStarted{},
  AnswerRecorded{answer}, GroundingScored{grounding}, QueryFailed{reason}.
  Commands: submit, attachSanitizedChunks, markAnswering, recordAnswer, recordGrounding,
  fail, getQuery. emptyState() returns Query.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state
  and Optional.of(...) inside the event-applier.

- 1 Consumer ChunkSanitizer subscribed to QueryEntity events; on QuerySubmitted runs
  a pipeline over every chunk's content: regex stages for API key patterns
  (sk-live-*, Bearer token-like strings), private-key PEM headers, connection strings
  with passwords, certificate fingerprints, and generic high-entropy token heuristic.
  Replaces matches with category-specific tokens ([REDACTED-API-KEY],
  [REDACTED-PEM-KEY], [REDACTED-CONN-STRING], [REDACTED-TOKEN]).
  Builds SanitizedChunks carrying the redacted chunk list and the categories found.
  Calls QueryEntity.attachSanitizedChunks(sanitized). After attachSanitizedChunks
  lands, the same Consumer starts a QueryWorkflow with id = "query-" + queryId.

- 1 View ChunkIndexView with row type QueryRow (mirrors Query minus
  request.retrievedChunks[*].content — raw chunk bodies stay on the entity for audit;
  the view holds chunk metadata for the UI). Table updater consumes QueryEntity events.
  ONE query getAllQueries: SELECT * AS queries FROM chunk_index_view. No WHERE status
  filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body
    {questionText, corpusTag, retrievedChunks: [{chunkId, filePath, startLine,
    endLine, language, content, corpusTag}], submittedBy}; mints queryId; calls
    QueryEntity.submit; returns {queryId}), GET /queries (list from getAllQueries,
    sorted newest-first), GET /queries/{id} (one row), GET /queries/sse (Server-Sent
    Events forwarded from the view's stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_CODE_QUESTION =
  Task.name("Answer code question").description("Read the attached code chunks and
  return a SearchAnswer with file paths and line numbers").resultConformsTo(
  SearchAnswer.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records CodeChunk, QueryRequest, SanitizedChunks, CodeReference, SearchAnswer,
  GroundingResult, Query, QueryStatus.

- GroundingScorer.java — pure deterministic logic (no LLM). Inputs: SearchAnswer and
  the list of SanitizedChunks. Outputs: GroundingResult. Scoring rubric documented in
  Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9276 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  CodeSearchAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern.

- src/main/resources/sample-events/corpus.jsonl with 20 seeded code chunks drawn from
  three corpus tags: akka-http (8 chunks covering route DSL, server binding, HTTP
  client), akka-streams (7 chunks covering Flow, Source, Sink, materializer), and
  akka-persistence (5 chunks covering EventSourcedEntity, snapshot, recovery). Each
  chunk has a realistic filePath (e.g., "src/main/java/com/example/
  HttpServerApp.java"), startLine/endLine, language ("java" or "scala"), and ~20 lines
  of plausible source content. Three chunks (one per tag) include a realistic but
  obviously synthetic secret string (e.g., private-key PEM header, bearer token) so
  the sanitizer has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (S1) matching Section 8.
  Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  data.data_classes.source-code = true, decisions.authority_level = informational-only
  (the agent's answer is a navigation aid, not an authoritative decision),
  oversight.human_in_loop = true (a developer reads the answer before acting),
  failure.failure_modes including "hallucinated-file-path", "stale-index",
  "secret-leakage-via-llm", "ungrounded-citation"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/code-search-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: CodeSearch Demo Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = question form + live list of query cards; right = selected-query detail with
  question text, sanitized chunk metadata list, answer paragraph, reference table, and
  grounding-score chip). Browser title exactly:
  <title>Akka Sample: CodeSearch Demo Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using
        the matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime. The message must not echo any
  captured key material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-code-question.json — 6 SearchAnswer entries covering questions about each
      corpus tag. Each entry has an answerText paragraph and a `references` array with
      1–3 CodeReference entries, each with a filePath matching a corpus chunk, valid
      startLine/endLine, language, and a 1-sentence relevanceBlurb. Plus 2 deliberately
      UNGROUNDED entries (file paths that are not in the seeded corpus, e.g.
      "src/main/java/com/example/NotAFile.java") — these exercise the GroundingScorer
      low-score path. The mock selects an ungrounded entry on the first iteration of
      every 4th query (modulo seed) so J3 is reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CodeSearchAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (answerStep 60s, awaitSanitizedStep 15s, groundingStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: QueryTasks.java with ANSWER_CODE_QUESTION = Task.name(...)
  .description(...).resultConformsTo(SearchAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9276 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel"> elements —
  Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CodeSearchAgent).
  The grounding check is rule-based (GroundingScorer.java) and does NOT make an LLM
  call — keeping the pattern's "one agent" promise honest.
- Code chunks are passed as task ATTACHMENTS, never inlined into the agent's
  instructions. Verify the generated answerStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
