# SPEC — kb-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** KbAgent.
**One-line pitch:** A user asks a natural-language question; one AI agent retrieves relevant passages from a Knowledge Base, reads them, and returns a grounded answer that cites only what was found — no hallucinated sources.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `KbAnswerAgent` (AutonomousAgent) carries the entire decision; the surrounding components handle retrieval, lifecycle tracking, and post-decision evaluation. One governance mechanism is wired around the agent:

- An **on-decision-eval** runs immediately after each `AnswerRecorded` event, scoring the answer for groundedness — every sentence in the answer must trace back to at least one of the retrieved passages. A score of 1–5 is emitted; low-scoring answers are flagged in the UI so a human can follow up.

The blueprint shows that a single-agent retrieval system can surface its own reliability signal without adding a second model call. The evaluator is deterministic and rule-based, keeping the "one agent" count honest.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** textarea (or picks one of three seeded example questions from a dropdown).
2. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~0.5 s, it transitions to `RETRIEVING` — the retriever ran, passage count is visible.
4. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWER_RECORDED`. The answer appears: a prose response and a citations list (passage id, document title, excerpt snippet).
5. Within ~1 s of the answer, the `evalStep` finishes. The card shows a **groundedness score** chip (1–5) plus a one-line rationale.
6. The user can ask another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `KbView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → retrieving → answering → answer_recorded → evaluated. Source of truth. | `QueryEndpoint`, `PassageRetriever`, `QueryWorkflow` | `KbView` |
| `PassageRetriever` | `Consumer` | Subscribes to `QuerySubmitted` events; runs in-process TF-IDF retrieval over the indexed Knowledge Base; calls `QueryEntity.attachPassages`. | `QueryEntity` events | `QueryEntity` |
| `KbDocumentConsumer` | `Consumer` | Subscribes to `DocumentIndexed` events; updates the in-process inverted index held in a singleton `KbIndex`. | `QueryEndpoint` (document ingestion) | `KbIndex` (in-process) |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitPassagesStep` → `answerStep` → `evalStep`. | started by `PassageRetriever` once passages attached | `KbAnswerAgent`, `QueryEntity` |
| `KbAnswerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question and retrieved passages as a task attachment; returns `KbAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `KbView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record KbDocument(
    String documentId,
    String title,
    String body,
    String collection,
    Instant indexedAt
) {}

record Passage(
    String passageId,
    String documentId,
    String documentTitle,
    String excerpt,
    double relevanceScore
) {}

record KbAnswer(
    String answer,
    List<Citation> citations,
    AnswerStatus answerStatus,
    Instant answeredAt
) {}
enum AnswerStatus { GROUNDED, NOT_FOUND, PARTIAL }

record Citation(
    String passageId,
    String documentTitle,
    String excerpt
) {}

record GroundednessResult(
    int score,           // 1..5
    String rationale,
    int citedSentences,
    int totalSentences,
    Instant evaluatedAt
) {}

record Query(
    String queryId,
    String questionText,
    String submittedBy,
    Optional<List<Passage>> passages,
    Optional<KbAnswer> answer,
    Optional<GroundednessResult> groundedness,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, RETRIEVING, ANSWERING, ANSWER_RECORDED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `PassagesAttached`, `AnsweringStarted`, `AnswerRecorded`, `GroundednessScored`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ questionText, submittedBy }` → `{ queryId }`.
- `POST /api/documents` — body `{ documentId, title, body, collection }` → `{ documentId }` (indexes a new document into the Knowledge Base).
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: KbAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + groundedness chip + age) and a right pane with the selected query's detail — question text, retrieved passages list, answer prose, citations table, and groundedness score chip with rationale.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision eval** (`eval-event`, `on-decision-eval`): runs immediately after `AnswerRecorded` lands, as `evalStep` inside the workflow. A deterministic rule-based `GroundednessScorer` (no LLM call — the eval is rule-based on purpose, so the same answer always scores the same) checks what fraction of the answer's sentences contain a phrase from one of the retrieved passages, whether every citation's `passageId` appears in the attached passage list, and whether the answer correctly identifies a NOT_FOUND case when zero passages were retrieved. Emits `GroundednessScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `KbAnswerAgent` → `prompts/kb-answer-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached passages, synthesise an answer to the question, and return one `Citation` per passage it drew on.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User asks a question covered by the seeded Knowledge Base; within 30 s the answer appears with at least one citation and a groundedness score ≥ 3.
2. **J2** — User asks a question with no matching passages; the agent returns an answer with `answerStatus = NOT_FOUND`; the groundedness scorer accepts this and scores it 5 (correct refusal is fully grounded behaviour).
3. **J3** — An answer whose sentences share no overlap with retrieved passages scores ≤ 2; the UI flags the card border red.
4. **J4** — A new document is ingested via `POST /api/documents`; a subsequent question covering that document's content retrieves passages from it and produces a grounded answer.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named kb-agent demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-kb-agent. Java package io.akka.samples.agentwithknowledgebase.
Akka 3.6.0. HTTP port 9827.

Components to wire (exactly):

- 1 AutonomousAgent KbAnswerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/kb-answer-agent.md>) and
  .capability(TaskAcceptance.of(KbTasks.ANSWER_QUESTION).maxIterationsPerTask(3)).
  The task receives the question text as its instruction text and the retrieved
  passages as a task ATTACHMENT named "passages.json" containing a JSON array of
  Passage records (NOT inlined into the instruction string — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call).
  Output: KbAnswer{answer: String, citations: List<Citation>,
  answerStatus: AnswerStatus (GROUNDED/NOT_FOUND/PARTIAL), answeredAt: Instant}.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitPassagesStep — polls QueryEntity.getQuery every 1s; on
    query.passages().isPresent() advances to answerStep.
    WorkflowSettings.stepTimeout 15s.
  * answerStep — emits AnsweringStarted, then calls
    componentClient.forAutonomousAgent(KbAnswerAgent.class, "answerer-" + queryId)
    .runSingleTask(
      TaskDef.instructions("Answer this question: " + query.questionText)
        .attachment("passages.json", serializePassages(query.passages).getBytes())
    ) — returns a taskId, then forTask(taskId).result(KbTasks.ANSWER_QUESTION) to
    fetch the answer. On success calls QueryEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery
    maxRetries(2).failoverTo(QueryWorkflow::error).
  * evalStep — runs a deterministic rule-based GroundednessScorer (NOT an LLM
    call) over the recorded answer and the attached passages: checks sentence-level
    overlap between answer text and passage excerpts, validates that every
    Citation.passageId is in the attached list, and awards full marks for a correct
    NOT_FOUND response when passages list is empty. Emits
    GroundednessScored{score: 1-5, rationale: String, citedSentences: int,
    totalSentences: int}. WorkflowSettings.stepTimeout 5s.
    error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a
  top-level WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State
  Query{queryId: String, questionText: String, submittedBy: String,
  passages: Optional<List<Passage>>, answer: Optional<KbAnswer>,
  groundedness: Optional<GroundednessResult>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}.
  QueryStatus enum: SUBMITTED, RETRIEVING, ANSWERING, ANSWER_RECORDED,
  EVALUATED, FAILED.
  Events: QuerySubmitted{questionText, submittedBy}, PassagesAttached{passages},
  AnsweringStarted{}, AnswerRecorded{answer}, GroundednessScored{groundedness},
  QueryFailed{reason}.
  Commands: submit, attachPassages, markAnswering, recordAnswer, recordGroundedness,
  fail, getQuery.
  emptyState() returns Query.initial("") with all Optional fields as
  Optional.empty() and status = SUBMITTED (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside event-applier.

- 1 Consumer PassageRetriever subscribed to QueryEntity events; on QuerySubmitted
  runs in-process TF-IDF retrieval over the KbIndex singleton (top-k = 5), builds
  a List<Passage>, then calls QueryEntity.attachPassages(passages). After
  attachPassages lands, the same Consumer starts a QueryWorkflow with id =
  "query-" + queryId.

- 1 Consumer KbDocumentConsumer subscribed to DocumentIndexed events emitted by
  QueryEndpoint when a document is POSTed; on DocumentIndexed tokenises the
  document body and upserts the TF-IDF index in KbIndex.

- 1 View KbView with row type QueryRow (mirrors Query minus full passage body
  text — the view holds passage excerpts only). Table updater consumes
  QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM kb_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with:
      POST /queries (body {questionText, submittedBy}; mints queryId; calls
        QueryEntity.submit; returns {queryId}),
      POST /documents (body {documentId, title, body, collection}; emits
        DocumentIndexed event; returns {documentId}),
      GET /queries (list from getAllQueries, sorted newest-first),
      GET /queries/{id} (one row),
      GET /queries/sse (Server-Sent Events forwarded from the view's
        stream-updates),
      and three /api/metadata/* endpoints serving the YAML/MD files from
      src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
      static-resources/*.

Companion files:

- KbTasks.java declaring one Task<R> constant:
    ANSWER_QUESTION = Task.name("Answer question")
      .description("Read the attached passages and produce a KbAnswer to the question")
      .resultConformsTo(KbAnswer.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class
  (Lesson 7).

- KbIndex.java — singleton in-process inverted index. Supports two operations:
  upsert(KbDocument) and retrieve(String question, int topK) -> List<Passage>.
  Implementation uses TF-IDF cosine similarity over unigrams. No external
  library required — a plain Java Map<String, Map<String, Double>> is
  sufficient for the seeded corpus size.

- GroundednessScorer.java — pure deterministic logic (no LLM). Inputs: KbAnswer
  and List<Passage>. Outputs: GroundednessResult. Scoring rubric:
    5 — every answer sentence overlaps at least one passage; all citations valid;
        or answer is correctly NOT_FOUND with empty passages.
    4 — ≥ 80% sentences grounded; all citations valid.
    3 — ≥ 60% sentences grounded; all citations valid.
    2 — < 60% grounded OR any citation references a missing passageId.
    1 — answer text shares no overlap with any passage (fully hallucinated).
  Documented in Javadoc on the class.

- Domain records: KbDocument, Passage, KbAnswer, Citation, AnswerStatus,
  GroundednessResult, Query, QueryStatus.

- src/main/resources/application.conf with
    akka.javasdk.dev-mode.http-port = 9827
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  KbAnswerAgent.definition() binds the configured provider via the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/kb-documents.jsonl with 12 seeded Knowledge
  Base documents across three collections: "product-faq" (5 docs), "policies" (4
  docs), "release-notes" (3 docs). Each document is 150–400 words.

- src/main/resources/sample-events/seed-questions.jsonl with 6 example questions:
  2 from product-faq, 2 from policies, 1 from release-notes, 1 with no good answer
  in the corpus (exercises the NOT_FOUND path).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching the mechanism
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with the values from Section 5 pre-filled.
  Deployer-specific fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/kb-answer-agent.md loaded as the agent system prompt.

- README.md at the project root matching the blueprint README.

- src/main/resources/static-resources/index.html — single self-contained file
  (no ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a
  two-column layout (left = live list of query cards; right = selected-query
  detail with question, passages list, answer prose, citations table, and
  groundedness score chip). Browser title exactly:
  <title>Akka Sample: KbAgent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing ModelProvider with per-task dispatch
  on Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly
  per call (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 6 KbAnswer entries:
      4 GROUNDED entries — each has a 2–4 sentence answer with phrases drawn from
        the seeded passages, and 1–3 Citations each referencing a real passageId
        from the seeded corpus.
      1 NOT_FOUND entry — answer explains the topic is not in the Knowledge Base,
        citations is empty list, answerStatus = NOT_FOUND.
      1 PARTIAL entry — answer synthesises some information but cites a passageId
        that does not exist in the retrieved list (exercises the groundedness
        scorer's citation-validation check, producing score ≤ 2).
- MockModelProvider.seedFor(queryId) makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. KbAnswerAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion KbTasks.java
  MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (answerStep 60s, awaitPassagesStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>.
- Lesson 7: KbTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(KbAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9827 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (KbAnswerAgent). The
  groundedness evaluator is rule-based (GroundednessScorer.java) and does NOT make
  an LLM call.
- The passages are passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated answerStep uses TaskDef.attachment(...).
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
