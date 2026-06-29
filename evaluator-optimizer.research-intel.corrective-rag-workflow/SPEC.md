# SPEC — corrective-rag-workflow

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Corrective RAG Workflow.
**One-line pitch:** Submit a research query; a retriever fetches document chunks; a relevance evaluator judges whether they are sufficient; if not, a web-search fallback supplements the context before an answer synthesizer produces the final response.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow sequences a retriever agent, a relevance evaluator agent, an optional web-search fallback agent, and an answer synthesizer agent, feeding each stage's output into the next. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every retrieval cycle's relevance verdict for downstream quality measurement, and a **guardrail** that gates the web-search tool call against a set of disallowed query patterns before the fallback fires.

## 3. User-facing flows

The user opens the App UI tab and submits a research query (a question string plus an optional relevance threshold override).

1. The system creates a `Query` record in `RETRIEVING` and starts a `CorrectiveRagWorkflow`.
2. The Retriever fetches candidate chunks from `CorpusStore` that match the query.
3. The Relevance Evaluator scores each chunk against the query (0–1 per chunk) and returns an overall `RelevanceVerdict`: `SUFFICIENT` when the top-k average meets the threshold, `INSUFFICIENT` otherwise.
4. If `SUFFICIENT`, the workflow skips the fallback and calls the Answer Synthesizer with the retrieved chunks as context.
5. If `INSUFFICIENT`, the web-search guardrail runs first. It checks the query against a set of disallowed patterns (e.g., personal-data lookups). A blocked query transitions to `ANSWER_DEGRADED` with the best available retrieved context rather than calling the external tool. A cleared query transitions to `WEB_SEARCHING`, calls the Web Search Agent, and collects `WebResult` entries.
6. After web search, the Answer Synthesizer generates the final answer from the combined context (retrieved chunks + web results), emitting `AnswerGenerated`.
7. The workflow transitions the query to `ANSWERED`, preserving all retrieved chunks, all relevance scores, all web results (if any), and the final answer on the entity.

If the workflow exceeds `maxRetrievalAttempts` (default 3) without a `SUFFICIENT` verdict and the guardrail blocks the fallback, the halt mechanism activates: the query lands in `ANSWER_DEGRADED` with the highest-scoring retrieval's context and a structured degradation reason.

A `QuerySimulator` (TimedAction) drips a canned query every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RetrieverAgent` | `AutonomousAgent` | Fetches ranked document chunks from `CorpusStore` for a query string. | `CorrectiveRagWorkflow` | returns `RetrievalResult` to workflow |
| `RelevanceEvaluatorAgent` | `AutonomousAgent` | Scores retrieved chunks against the query; returns `SUFFICIENT` or `INSUFFICIENT` with per-chunk scores. | `CorrectiveRagWorkflow` | returns `RelevanceVerdict` to workflow |
| `WebSearchAgent` | `AutonomousAgent` | Executes a web-search fallback when retrieval is insufficient; returns ranked `WebResult` entries. | `CorrectiveRagWorkflow` | returns `WebSearchResult` to workflow |
| `AnswerSynthesizerAgent` | `AutonomousAgent` | Generates the final answer from query + assembled context; cites sources. | `CorrectiveRagWorkflow` | returns `Answer` to workflow |
| `CorrectiveRagWorkflow` | `Workflow` | Orchestrates retrieve → evaluate → (guardrail + fallback?) → synthesize; halts at ceiling. | `RagEndpoint`, `QueryConsumer` | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Holds the query lifecycle, every retrieval, every relevance verdict, web results, and the final answer. | `CorrectiveRagWorkflow` | `QueryView` |
| `CorpusStore` | `EventSourcedEntity` | In-process document corpus; answers chunk-fetch commands by keyword matching. | `RetrieverAgent` (via workflow call) | — |
| `QueryView` | `View` | List-of-queries read model. | `QueryEntity` events | `RagEndpoint` |
| `QueryConsumer` | `Consumer` | Subscribes to `QueryEntity` `QueryCreated` events; starts a workflow per new query. | `QueryEntity` events | `CorrectiveRagWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample query every 60 s from `sample-events/research-queries.jsonl`. | scheduler | `RagEndpoint` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `QueryView`, records an `EvalRecorded` event for any retrieval cycle completed since the last tick. | scheduler | `QueryEntity` |
| `RagEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `QueryView`, `QueryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record QueryRequest(String question, Optional<Double> relevanceThresholdOverride, String requestedBy) {}

record ChunkScore(String chunkId, String excerpt, double score) {}

record RetrievalResult(List<ChunkScore> chunks, Instant retrievedAt) {}

record RelevanceVerdict(
    RelevanceJudgment judgment,
    List<ChunkScore> scoredChunks,
    double averageScore,
    String rationale,
    Instant evaluatedAt
) {}

record WebResult(String url, String title, String snippet, double relevanceScore) {}

record WebSearchResult(List<WebResult> results, String searchQuery, Instant searchedAt) {}

record GuardrailVerdict(boolean cleared, String reasonCode, String detail) {}

record Answer(
    String text,
    List<String> citedChunkIds,
    List<String> citedUrls,
    AnswerSource source,
    Instant generatedAt
) {}

record RetrievalAttempt(
    int attemptNumber,
    RetrievalResult retrieval,
    RelevanceVerdict verdict,
    Optional<GuardrailVerdict> guardrail,
    Optional<WebSearchResult> webSearch
) {}

record Query(
    String queryId,
    String question,
    double relevanceThreshold,
    int maxRetrievalAttempts,
    QueryStatus status,
    List<RetrievalAttempt> attempts,
    Optional<Answer> answer,
    Optional<String> degradationReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus { RETRIEVING, EVALUATING, WEB_SEARCHING, ANSWERED, ANSWER_DEGRADED }

enum RelevanceJudgment { SUFFICIENT, INSUFFICIENT }

enum AnswerSource { RETRIEVAL_ONLY, RETRIEVAL_PLUS_WEB, DEGRADED_RETRIEVAL }
```

### Events (on `QueryEntity`)

`QueryCreated`, `RetrievalAttempted`, `RelevanceVerdictRecorded`, `GuardrailVerdictRecorded`, `WebSearchCompleted`, `AnswerGenerated`, `QueryAnswered`, `QueryDegraded`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/queries` — body `{ question, relevanceThresholdOverride?, requestedBy? }` → `{ queryId }`. Starts a workflow.
- `GET /api/queries` — list all queries. Optional `?status=RETRIEVING|EVALUATING|WEB_SEARCHING|ANSWERED|ANSWER_DEGRADED`.
- `GET /api/queries/{id}` — one query (including every retrieval attempt, every verdict, and the final answer).
- `GET /api/queries/sse` — server-sent events stream of every query change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Corrective RAG Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a query, live list of queries with status pills, click-to-expand per-attempt timeline showing each retrieval, the relevance verdict, the guardrail verdict (if triggered), web results (if any), and the final answer with source attribution.

Browser title: `<title>Akka Sample: Corrective RAG Workflow</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-event** (`on-decision-eval`): every retrieval cycle's relevance verdict is recorded as an `EvalRecorded` event with `{ attemptNumber, judgment, averageScore, webSearchFired }`. The `EvalSampler` TimedAction is the canonical writer; the workflow itself also emits one event on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-attempt timeline and in `/api/queries/{id}`.
- **H1 — guardrail** (`before-tool-call`): before the Web Search Agent is called, the query is checked against a set of disallowed patterns (personal-data identifiers, blocked domains). A blocked query does not proceed to the external tool call; instead the workflow transitions to `ANSWER_DEGRADED` with the best available retrieved context. Enforcement: blocking.

## 9. Agent prompts

- `RetrieverAgent` → `prompts/retriever.md`. Given a query string and a reference to `CorpusStore`, returns ranked chunks.
- `RelevanceEvaluatorAgent` → `prompts/relevance-evaluator.md`. Scores each chunk 0–1; returns overall judgment and rationale.
- `WebSearchAgent` → `prompts/web-search.md`. Executes the search, returns ranked `WebResult` entries.
- `AnswerSynthesizerAgent` → `prompts/answer-synthesizer.md`. Generates a cited answer from the assembled context.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — retrieval-sufficient path** — Submit a query whose topic is covered by the in-process corpus; the query progresses `RETRIEVING` → `EVALUATING` → `ANSWERED` with `AnswerSource = RETRIEVAL_ONLY`; no web search fires.
2. **J2 — corrective fallback path** — Submit a query with no corpus coverage; the system detects `INSUFFICIENT`, the guardrail clears, web search fires, and the final answer is `RETRIEVAL_PLUS_WEB`; the expanded view shows every chunk score, the `INSUFFICIENT` verdict, the web results, and the synthesized answer.
3. **J3 — guardrail block** — Submit a query containing a disallowed pattern; the guardrail records `cleared = false` with `reasonCode = DISALLOWED_PATTERN`; the web-search agent is never called; the query transitions to `ANSWER_DEGRADED`; the entity preserves the blocked query and the reason.
4. **J4 — eval-event timeline** — The expanded view of any completed query shows one `EvalRecorded` event per retrieval attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named corrective-rag-workflow demonstrating the evaluator-optimizer ×
research-intel cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-research-intel-corrective-rag-workflow.
Java package io.akka.samples.correctiveragworkflow. Akka 3.6.0. HTTP port 9220.

Components to wire (exactly):
- 4 AutonomousAgents:
  * RetrieverAgent — definition() with
    capability(TaskAcceptance.of(RETRIEVE).maxIterationsPerTask(2)).
    System prompt loaded from prompts/retriever.md. Returns
    RetrievalResult{chunks, retrievedAt}.
  * RelevanceEvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE_RELEVANCE).maxIterationsPerTask(2)).
    System prompt from prompts/relevance-evaluator.md. Returns
    RelevanceVerdict{judgment, scoredChunks, averageScore, rationale, evaluatedAt}
    where judgment is the RelevanceJudgment enum (SUFFICIENT | INSUFFICIENT).
  * WebSearchAgent — definition() with
    capability(TaskAcceptance.of(WEB_SEARCH).maxIterationsPerTask(2)).
    System prompt from prompts/web-search.md. Returns
    WebSearchResult{results, searchQuery, searchedAt}.
  * AnswerSynthesizerAgent — definition() with
    capability(TaskAcceptance.of(SYNTHESIZE).maxIterationsPerTask(2)).
    System prompt from prompts/answer-synthesizer.md. Returns
    Answer{text, citedChunkIds, citedUrls, source, generatedAt}.

- 1 Workflow CorrectiveRagWorkflow with steps:
    startStep -> retrieveStep -> evaluateStep ->
    [SUFFICIENT? synthesizeStep : guardrailStep] ->
    [guardrail CLEARED? webSearchStep -> synthesizeStep : degradeStep] ->
    END.
  retrieveStep calls forAutonomousAgent(RetrieverAgent.class, queryId)
    .runSingleTask(RETRIEVE) then forTask(taskId).result(RETRIEVE).
  evaluateStep calls forAutonomousAgent(RelevanceEvaluatorAgent.class, queryId)
    .runSingleTask(EVALUATE_RELEVANCE). If INSUFFICIENT and attemptNumber <
    maxRetrievalAttempts, loop back to retrieveStep with a nudge note; if
    INSUFFICIENT and attempts exhausted or guardrail blocks, degradeStep.
  guardrailStep is a pure-function step (no LLM call): checks the query
    against the disallowed-pattern list loaded from application.conf
    (corrective-rag.guardrail.disallowed-patterns). On BLOCKED, emits
    GuardrailVerdictRecorded with cleared=false and reasonCode="DISALLOWED_PATTERN",
    then transitions to degradeStep. On CLEARED, emits GuardrailVerdictRecorded
    with cleared=true and transitions to webSearchStep.
  webSearchStep calls forAutonomousAgent(WebSearchAgent.class, queryId)
    .runSingleTask(WEB_SEARCH).
  synthesizeStep calls forAutonomousAgent(AnswerSynthesizerAgent.class, queryId)
    .runSingleTask(SYNTHESIZE); emits AnswerGenerated then QueryAnswered.
  degradeStep emits QueryDegraded with the best-scoring chunk's excerpt and a
    structured degradationReason.
  Override settings() with stepTimeout(60s) on retrieveStep, evaluateStep,
    webSearchStep, and synthesizeStep.
    defaultStepRecovery(maxRetries(2).failoverTo(degradeStep)).

- 1 EventSourcedEntity QueryEntity holding state Query{queryId, question,
  relevanceThreshold, maxRetrievalAttempts, QueryStatus status,
  List<RetrievalAttempt> attempts, Optional<Answer> answer,
  Optional<String> degradationReason, Instant createdAt,
  Optional<Instant> finishedAt}.
  QueryStatus enum: RETRIEVING, EVALUATING, WEB_SEARCHING, ANSWERED, ANSWER_DEGRADED.
  RelevanceJudgment enum: SUFFICIENT, INSUFFICIENT.
  AnswerSource enum: RETRIEVAL_ONLY, RETRIEVAL_PLUS_WEB, DEGRADED_RETRIEVAL.
  Events: QueryCreated, RetrievalAttempted, RelevanceVerdictRecorded,
    GuardrailVerdictRecorded, WebSearchCompleted, AnswerGenerated,
    QueryAnswered, QueryDegraded, EvalRecorded.
  Commands: createQuery, recordRetrieval, recordRelevance, recordGuardrail,
    recordWebSearch, recordAnswer, answerQuery, degradeQuery, recordEval,
    getQuery.
  emptyState() returns Query.initial("", "", 0.7, 3) with no
    commandContext() reference. Event-applier wraps lifecycle fields with
    Optional.of(...).

- 1 EventSourcedEntity CorpusStore holding a map of chunkId → DocumentChunk
  (String chunkId, String content, List<String> keywords). Answers
  fetchChunks(query) by returning the top-5 chunks whose keywords overlap
  the query tokens, scored by overlap count. Pre-seeded via
  CorpusSeeder (TimedAction or Bootstrap init) that reads
  src/main/resources/sample-events/corpus-chunks.jsonl.

- 1 View QueryView with row type QueryRow (mirrors Query; the attempts list
  is preserved as-is — bounded at maxRetrievalAttempts). Table updater
  consumes QueryEntity events. ONE query getAllQueries SELECT * AS queries
  FROM queries_view. No WHERE status filter — caller filters client-side
  because Akka cannot auto-index enum columns (Lesson 2).

- 1 Consumer QueryConsumer subscribed to QueryEntity QueryCreated events; on
  QueryCreated starts a CorrectiveRagWorkflow with the queryId as the
  workflow id.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-queries.jsonl and calls
    RagEndpoint POST /api/queries.
  * EvalSampler — every 30s, queries QueryView.getAllQueries, finds queries
    with a completed relevance verdict that has not yet been recorded as an
    EvalRecorded event, and calls QueryEntity.recordEval(attemptNumber,
    judgment, averageScore, webSearchFired). Idempotent per (queryId, attemptNumber).

- 2 HttpEndpoints:
  * RagEndpoint at /api with POST /queries, GET /queries, GET /queries/{id},
    GET /queries/sse, and three /api/metadata/* endpoints serving the
    YAML/MD files from src/main/resources/metadata/. The POST /queries body
    is {question, relevanceThresholdOverride?, requestedBy?}; missing
    relevanceThresholdOverride defaults to corrective-rag.retrieval.default-threshold
    (0.7), missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- RagTasks.java declaring four Task<R> constants: RETRIEVE (resultConformsTo
  RetrievalResult), EVALUATE_RELEVANCE (RelevanceVerdict), WEB_SEARCH
  (WebSearchResult), SYNTHESIZE (Answer).
- Domain records QueryRequest, ChunkScore, RetrievalResult, RelevanceVerdict,
  WebResult, WebSearchResult, GuardrailVerdict, Answer, RetrievalAttempt, Query;
  enums QueryStatus, RelevanceJudgment, AnswerSource.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9220
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  the canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  corrective-rag.retrieval.max-attempts = 3,
  corrective-rag.retrieval.default-threshold = 0.7,
  corrective-rag.guardrail.disallowed-patterns = ["\\bSSN\\b","\\bpassport\\b","\\bcredit.card\\b"],
  overridable by env var.
- src/main/resources/sample-events/research-queries.jsonl with 8 canned query
  lines, each shaped {"question":"...", "requestedBy":"simulator"}.
- src/main/resources/sample-events/corpus-chunks.jsonl with 20 document chunk
  lines, each shaped {"chunkId":"...", "content":"...", "keywords":[...]}.
  Topics drawn from research-intel subjects (regulatory frameworks, AI policy,
  governance standards, research methodology).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-event
  on-decision-eval, H1 guardrail before-tool-call) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = research-query-answering,
  decisions.authority_level = informational-only,
  data.data_classes.pii = false, capabilities.web-search = true;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/retriever.md, prompts/relevance-evaluator.md, prompts/web-search.md,
  prompts/answer-synthesizer.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Corrective RAG Workflow",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO Configuration
  section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview (eyebrow + headline + no subtitle + Try
  it / How it works / Components / API contract cards); Architecture
  (4 mermaid diagrams + click-to-expand component table); Risk Survey (7
  sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45); Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows); App UI (form + live list with status pills, click-to-expand
  per-attempt timeline). Browser title exactly:
  <title>Akka Sample: Corrective RAG Workflow</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory;
        gone when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class name
  (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json, picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    retriever.json — 6 RetrievalResult entries. Three return 3–5 chunks
      with scores 0.6–0.9 on topics present in corpus-chunks.jsonl. Two
      return 1–2 chunks with scores 0.2–0.4 (low-match scenario to
      trigger INSUFFICIENT). One returns an empty chunks list (zero-match
      scenario).
    relevance-evaluator.json — 6 RelevanceVerdict entries. Three return
      judgment=SUFFICIENT with averageScore >= 0.7. Three return
      judgment=INSUFFICIENT with averageScore < 0.7 and a rationale bullet.
    web-search.json — 4 WebSearchResult entries with 3–5 WebResult rows
      each, varying relevanceScore 0.5–0.95.
    answer-synthesizer.json — 6 Answer entries with text 80–300 chars,
      citedChunkIds and citedUrls from corpus and web-search mock data,
      source alternating RETRIEVAL_ONLY / RETRIEVAL_PLUS_WEB /
      DEGRADED_RETRIEVAL.
- A MockModelProvider.seedFor(queryId, attemptNumber) helper makes the
  selection deterministic per (queryId, attemptNumber).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. All four
  agents extend akka.javasdk.agent.autonomous.AutonomousAgent and ship
  with a RagTasks companion declaring the four Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Query row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: RagTasks.java is mandatory; generating any agent without it is
  a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9220, declared in application.conf dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. The DOM contains exactly five
  <section class="tab-panel"> elements; removed panels are deleted from
  the HTML, not hidden with display:none.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars and the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
