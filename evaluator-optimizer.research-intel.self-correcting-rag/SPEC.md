# SPEC — self-correcting-rag

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Adaptive/Corrective RAG.
**One-line pitch:** Submit a research question; a retriever agent fetches documents; a relevance grader filters them; if too few pass, the pipeline rewrites the query or switches to web search; a generator produces a grounded answer; a hallucination grader accepts it or sends the loop back.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to retrieval-augmented generation: a `RagWorkflow` orchestrates five agents across two evaluation stages (document relevance grading, hallucination grading), routing the pipeline through adaptive correction paths — query rewriting and web-search fallback — when retrieval quality is insufficient. The blueprint demonstrates two governance mechanisms: an **output guardrail** that gates the final answer against a hallucination check before surfacing it to the caller, and an **eval-event** that records each grading step's verdict for downstream quality measurement.

## 3. User-facing flows

The user opens the App UI tab and submits a research question.

1. The system creates a `Query` record in `RETRIEVING` and starts a `RagWorkflow`.
2. `RetrieverAgent` fetches the top-K candidate documents from `CorpusEntity` for the query.
3. `RelevanceGraderAgent` scores each document; documents below the threshold are marked `IRRELEVANT` and discarded.
4. If at least `minRelevantDocs` (default 1) documents are retained, the pipeline advances to generation.
5. If no documents clear the threshold, `QueryRewriterAgent` rewrites the query. The pipeline retries retrieval with the rewritten query. If the rewrite ceiling is reached without relevant documents, the query transitions to `FAILED_NO_RELEVANT_DOCS`.
6. `GeneratorAgent` produces a grounded answer using only the retained documents.
7. `HallucinationGraderAgent` checks whether the answer is supported by the retained documents. If `GROUNDED`, the workflow transitions to `ANSWERED`. If `HALLUCINATED`, the pipeline retries generation up to `maxGenerationAttempts` (default 2). If the ceiling is hit, the query transitions to `FAILED_HALLUCINATION`.
8. Each grading decision is emitted as an `EvalRecorded` event.

A `QuerySimulator` (TimedAction) drips a canned query every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RetrieverAgent` | `AutonomousAgent` | Fetches top-K candidate documents from `CorpusEntity` for the query. | `RagWorkflow` | returns `RetrievalResult` to workflow |
| `RelevanceGraderAgent` | `AutonomousAgent` | Scores each retrieved document for relevance; returns a list of `DocumentGrade` records. | `RagWorkflow` | returns `GradingResult` to workflow |
| `QueryRewriterAgent` | `AutonomousAgent` | Rewrites a query that produced no relevant documents; returns a `RewrittenQuery`. | `RagWorkflow` | returns `RewrittenQuery` to workflow |
| `GeneratorAgent` | `AutonomousAgent` | Produces a grounded `GeneratedAnswer` from retained documents. | `RagWorkflow` | returns `GeneratedAnswer` to workflow |
| `HallucinationGraderAgent` | `AutonomousAgent` | Checks whether the generated answer is supported by the retained documents; returns `HallucinationVerdict`. | `RagWorkflow` | returns `HallucinationVerdict` to workflow |
| `RagWorkflow` | `Workflow` | Orchestrates retrieve → grade → (rewrite|continue) → generate → hallucination-check loop. | `RagEndpoint`, `QueryConsumer` | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Holds the query lifecycle, every retrieval pass, every grading result, and the final answer. | `RagWorkflow` | `QueryView` |
| `CorpusEntity` | `EventSourcedEntity` | Stores the in-memory document corpus; answers fetch commands. | `RagWorkflow` (fetch), `RagEndpoint` (admin) | `RetrieverAgent` |
| `QueryView` | `View` | List-of-queries read model. | `QueryEntity` events | `RagEndpoint` |
| `QueryConsumer` | `Consumer` | Subscribes to `QueryEntity` events; starts a `RagWorkflow` per submitted query. | `QueryEntity` events | `RagWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample query every 60 s from `sample-events/research-queries.jsonl`. | scheduler | `QueryEntity` via `RagEndpoint` |
| `EvalSampler` | `TimedAction` | Every 30 s, scans `QueryView`, records an `EvalRecorded` event for any grading step completed since the last tick. | scheduler | `QueryEntity` |
| `RagEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, get, list, SSE; corpus admin; `/api/metadata/*`. | — | `QueryView`, `QueryEntity`, `CorpusEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record QueryRequest(String queryText, String requestedBy) {}

record Document(String docId, String title, String content, String source) {}

record DocumentGrade(String docId, RelevanceVerdict verdict, double score, String rationale) {}

record RetrievalResult(String queryUsed, List<Document> documents, Instant retrievedAt) {}

record GradingResult(List<DocumentGrade> grades, List<Document> retainedDocuments, int totalRetrieved, Instant gradedAt) {}

record RewrittenQuery(String originalQuery, String rewrittenQueryText, String rewriteRationale, Instant rewrittenAt) {}

record GeneratedAnswer(String answerText, List<String> citedDocIds, Instant generatedAt) {}

record HallucinationVerdict(HallucinationResult result, String rationale, List<String> unsupportedClaims, Instant gradedAt) {}

record RetrievalPass(
    int passNumber,
    RetrievalResult retrieval,
    GradingResult grading,
    Optional<RewrittenQuery> rewrite
) {}

record Query(
    String queryId,
    String queryText,
    String requestedBy,
    int maxRewriteAttempts,
    int maxGenerationAttempts,
    QueryStatus status,
    List<RetrievalPass> retrievalPasses,
    Optional<GeneratedAnswer> generatedAnswer,
    Optional<HallucinationVerdict> hallucinationVerdict,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus { RETRIEVING, GRADING, REWRITING, GENERATING, HALLUCINATION_CHECK, ANSWERED, FAILED_NO_RELEVANT_DOCS, FAILED_HALLUCINATION }

enum RelevanceVerdict { RELEVANT, IRRELEVANT }

enum HallucinationResult { GROUNDED, HALLUCINATED }
```

### Events (on `QueryEntity`)

`QueryCreated`, `RetrievalPassStarted`, `DocumentsRetrieved`, `DocumentsGraded`, `QueryRewritten`, `AnswerGenerated`, `HallucinationChecked`, `QueryAnswered`, `QueryFailedNoRelevantDocs`, `QueryFailedHallucination`, `EvalRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/queries` — body `{ queryText, requestedBy? }` → `{ queryId }`. Starts a workflow.
- `GET /api/queries` — list all queries. Optional `?status=RETRIEVING|GRADING|...`.
- `GET /api/queries/{id}` — one query (including every retrieval pass, every grade, and the final answer).
- `GET /api/queries/sse` — server-sent events stream of every query change.
- `POST /api/corpus/documents` — body `{ docId, title, content, source }` → `204`. Adds a document to `CorpusEntity`.
- `GET /api/corpus/documents` — list all corpus documents.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "Adaptive/Corrective RAG"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a query, live list of queries with status pills, click-to-expand per-pass timeline showing each retrieval, the grading results, and the final answer with hallucination verdict.

Browser title: `<title>Akka Sample: Adaptive/Corrective RAG</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`after-llm-response` on `GeneratorAgent`): `HallucinationGraderAgent` checks the generated answer against the retained documents before the answer is surfaced to the caller. A `HALLUCINATED` verdict routes the pipeline back to `GeneratorAgent` for re-generation (up to `maxGenerationAttempts`). If the ceiling is hit, the query transitions to `FAILED_HALLUCINATION`. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every relevance-grading step and every hallucination-grading step emits an `EvalRecorded` event with `{ passNumber, stepKind, verdict, score, docCount }`. The `EvalSampler` TimedAction is the canonical writer; the workflow also emits an event on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-pass timeline and in `/api/queries/{id}`.

## 9. Agent prompts

- `RetrieverAgent` → `prompts/retriever.md`. Issues a fetch command to `CorpusEntity` for the top-K documents; returns a `RetrievalResult`.
- `RelevanceGraderAgent` → `prompts/relevance-grader.md`. Scores each document against the query; returns a `GradingResult` with a `DocumentGrade` per document.
- `QueryRewriterAgent` → `prompts/query-rewriter.md`. Rewrites a query that produced no relevant documents; returns a `RewrittenQuery` with rationale.
- `GeneratorAgent` → `prompts/generator.md`. Produces a grounded answer from retained documents only; returns a `GeneratedAnswer` with cited doc IDs.
- `HallucinationGraderAgent` → `prompts/hallucination-grader.md`. Checks whether every claim in the answer is supported by the retained documents; returns a `HallucinationVerdict`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — end-to-end answer** — Submit a query; pipeline progresses `RETRIEVING` → `GRADING` → `GENERATING` → `HALLUCINATION_CHECK` → `ANSWERED`; the App UI shows the grounded answer with cited document IDs.
2. **J2 — query rewrite path** — Submit a query whose documents all grade `IRRELEVANT`; the pipeline invokes `QueryRewriterAgent`, performs a second retrieval pass, and either answers or transitions to `FAILED_NO_RELEVANT_DOCS`.
3. **J3 — hallucination re-generation** — Submit a query where the first generated answer grades `HALLUCINATED`; the pipeline routes back to `GeneratorAgent`; the second answer grades `GROUNDED` and the query reaches `ANSWERED`.
4. **J4 — eval-event timeline** — The expanded view of any answered query shows one `EvalRecorded` event per grading step and one terminal eval event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named self-correcting-rag demonstrating the evaluator-optimizer ×
research-intel cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-research-intel-self-correcting-rag.
Java package io.akka.samples.adaptivecorrectiverag. Akka 3.6.0. HTTP port 9521.

Components to wire (exactly):
- 5 AutonomousAgents:
  * RetrieverAgent — definition() with
    capability(TaskAcceptance.of(RETRIEVE).maxIterationsPerTask(2)).
    System prompt loaded from prompts/retriever.md. Returns
    RetrievalResult{queryUsed, documents, retrievedAt}.
  * RelevanceGraderAgent — definition() with
    capability(TaskAcceptance.of(GRADE_RELEVANCE).maxIterationsPerTask(2)).
    System prompt from prompts/relevance-grader.md. Returns
    GradingResult{grades, retainedDocuments, totalRetrieved, gradedAt} where
    each DocumentGrade has verdict RelevanceVerdict (RELEVANT | IRRELEVANT).
  * QueryRewriterAgent — definition() with
    capability(TaskAcceptance.of(REWRITE_QUERY).maxIterationsPerTask(2)).
    System prompt from prompts/query-rewriter.md. Returns
    RewrittenQuery{originalQuery, rewrittenQueryText, rewriteRationale, rewrittenAt}.
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3)).
    System prompt from prompts/generator.md. Returns
    GeneratedAnswer{answerText, citedDocIds, generatedAt}.
  * HallucinationGraderAgent — definition() with
    capability(TaskAcceptance.of(GRADE_HALLUCINATION).maxIterationsPerTask(2)).
    System prompt from prompts/hallucination-grader.md. Returns
    HallucinationVerdict{result, rationale, unsupportedClaims, gradedAt}
    where result is HallucinationResult enum (GROUNDED | HALLUCINATED).

- 1 Workflow RagWorkflow with steps:
    startStep -> retrieveStep -> gradeRelevanceStep ->
    [noRelevantDocs? (passCount < maxRewriteAttempts ? rewriteStep -> retrieveStep
       : failNoDocsStep) : generateStep] ->
    hallucinationCheckStep ->
    [HALLUCINATED? (generationAttempt < maxGenerationAttempts ? generateStep
       : failHallucinationStep) : acceptStep] -> END.
  retrieveStep calls forAutonomousAgent(RetrieverAgent.class, queryId).runSingleTask(RETRIEVE).
  gradeRelevanceStep calls forAutonomousAgent(RelevanceGraderAgent.class, queryId).runSingleTask(GRADE_RELEVANCE).
  rewriteStep calls forAutonomousAgent(QueryRewriterAgent.class, queryId).runSingleTask(REWRITE_QUERY).
  generateStep calls forAutonomousAgent(GeneratorAgent.class, queryId).runSingleTask(GENERATE).
  hallucinationCheckStep calls forAutonomousAgent(HallucinationGraderAgent.class, queryId).runSingleTask(GRADE_HALLUCINATION).
  acceptStep emits QueryAnswered.
  failNoDocsStep emits QueryFailedNoRelevantDocs with failureReason.
  failHallucinationStep emits QueryFailedHallucination with failureReason.
  Override settings() with stepTimeout(60s) on retrieveStep, gradeRelevanceStep,
  rewriteStep, generateStep, hallucinationCheckStep.
  defaultStepRecovery(maxRetries(2).failoverTo(failHallucinationStep)).

- 1 EventSourcedEntity QueryEntity holding state Query{queryId, queryText,
  requestedBy, maxRewriteAttempts, maxGenerationAttempts, QueryStatus status,
  List<RetrievalPass> retrievalPasses, Optional<GeneratedAnswer> generatedAnswer,
  Optional<HallucinationVerdict> hallucinationVerdict, Optional<String> failureReason,
  Instant createdAt, Optional<Instant> finishedAt}.
  QueryStatus enum: RETRIEVING, GRADING, REWRITING, GENERATING, HALLUCINATION_CHECK,
  ANSWERED, FAILED_NO_RELEVANT_DOCS, FAILED_HALLUCINATION.
  Events: QueryCreated, RetrievalPassStarted, DocumentsRetrieved, DocumentsGraded,
  QueryRewritten, AnswerGenerated, HallucinationChecked, QueryAnswered,
  QueryFailedNoRelevantDocs, QueryFailedHallucination, EvalRecorded.
  Commands: createQuery, recordRetrieval, recordGrades, recordRewrite, recordAnswer,
  recordHallucinationCheck, answerQuery, failNoDocs, failHallucination, recordEval,
  getQuery.

- 1 EventSourcedEntity CorpusEntity with command addDocument(docId, title, content,
  source) emitting DocumentAdded{docId, title, content, source, addedAt} and
  command fetchDocuments(queryText, topK) returning List<Document>.

- 1 View QueryView with row type QueryRow (mirrors Query; the retrievalPasses list
  is bounded at maxRewriteAttempts so size stays reasonable). Table updater consumes
  QueryEntity events. ONE query getAllQueries SELECT * AS queries FROM queries_view.
  No WHERE status filter — caller filters client-side (Lesson 2).

- 1 Consumer QueryConsumer subscribed to QueryEntity events; on QueryCreated starts
  a RagWorkflow with the queryId as the workflow id.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/research-queries.jsonl and calls
    RagEndpoint POST /api/queries.
  * EvalSampler — every 30s, queries QueryView.getAllQueries, finds grading steps
    completed since the last tick that have not yet been recorded as EvalRecorded
    events, and calls QueryEntity.recordEval(passNumber, stepKind, verdict, score,
    docCount). Idempotent per (queryId, passNumber, stepKind).

- 2 HttpEndpoints:
  * RagEndpoint at /api with POST /queries, GET /queries, GET /queries/{id},
    GET /queries/sse, POST /corpus/documents, GET /corpus/documents, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /queries body is {queryText,
    requestedBy?}; missing requestedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- RagTasks.java declaring five Task<R> constants: RETRIEVE (resultConformsTo
  RetrievalResult), GRADE_RELEVANCE (GradingResult), REWRITE_QUERY (RewrittenQuery),
  GENERATE (GeneratedAnswer), GRADE_HALLUCINATION (HallucinationVerdict).
- Domain records QueryRequest, Document, DocumentGrade, RetrievalResult,
  GradingResult, RewrittenQuery, GeneratedAnswer, HallucinationVerdict,
  RetrievalPass, Query; enums QueryStatus, RelevanceVerdict, HallucinationResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9521 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  self-correcting-rag.retrieval.max-rewrite-attempts = 2 and
  self-correcting-rag.generation.max-generation-attempts = 2 and
  self-correcting-rag.retrieval.top-k = 5, overridable by env var.
- src/main/resources/sample-events/research-queries.jsonl with 8 canned query
  lines, each shaped {"queryText":"...", "requestedBy":"simulator"}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = research-question-answering,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/retriever.md, prompts/relevance-grader.md, prompts/query-rewriter.md,
  prompts/generator.md, prompts/hallucination-grader.md loaded at agent startup.
- README.md at the project root: title "Akka Sample: Adaptive/Corrective RAG",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section.
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
  per-pass timeline). Browser title exactly:
  <title>Akka Sample: Adaptive/Corrective RAG</title>.

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
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM via the MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning
        the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: retriever.json, relevance-grader.json,
  query-rewriter.json, generator.json, hallucination-grader.json), picks
  one entry pseudo-randomly per call, and deserialises it into the agent's
  typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    retriever.json — 5 RetrievalResult entries, each containing 3–5
      Document objects on different research topics matching the queries in
      research-queries.jsonl.
    relevance-grader.json — 6 GradingResult entries. Four pass all
      documents as RELEVANT with score 0.8–1.0. Two grade all documents as
      IRRELEVANT with score 0.1–0.2 (to exercise the rewrite path).
    query-rewriter.json — 4 RewrittenQuery entries with concise rewritten
      query text and a brief rewriteRationale.
    generator.json — 5 GeneratedAnswer entries with 2–4 sentence answers
      and 1–3 cited docIds.
    hallucination-grader.json — 6 HallucinationVerdict entries. Four return
      result=GROUNDED with empty unsupportedClaims. Two return
      result=HALLUCINATED with 1 unsupportedClaims entry (to exercise the
      re-generation path).
- A MockModelProvider.seedFor(queryId, passNumber) helper makes the
  selection deterministic per (queryId, passNumber) so the same query in
  dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. All five
  agents extend akka.javasdk.agent.autonomous.AutonomousAgent and ship with
  a RagTasks companion declaring the five Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the Query row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: RagTasks.java is mandatory; generating any agent without it is
  a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9521, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words (shape, minimal, smaller, complex, Akka SDK
  in narrative, marketing tone, competitor brand names) do not appear in
  any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND theme variables for state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc.
- Lesson 25: NEVER write the key value to disk. application.conf records
  only ${?VAR_NAME} substitution; Bootstrap.java fails fast if the
  reference does not resolve.
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
