# SPEC — rag-baseline

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RAG Sample.
**One-line pitch:** A user asks a natural-language question; one AI agent retrieves the most relevant passages from an in-process vector store (passed as task attachments, never as inline prompt text) and returns a grounded answer with per-claim citations and a 1–5 groundedness score.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `AnswerAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its retrieval context and audit its output. One governance mechanism is wired:

- An **on-decision evaluator** runs immediately after each `AnswerRecorded` event, scoring the answer for groundedness — does every claim in the answer map to a cited passage id from the retrieved set? Are citation ids fabricated? This eval is rule-based and deterministic; it does not make an LLM call.

The blueprint shows that the retrieval and evaluation responsibilities stay in ordinary Akka components — the agent is responsible only for reading the attached passages and producing a grounded answer in the required structure.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** input (or picks one of three seeded questions from a dropdown).
2. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `SUBMITTED` state. Within ~1 s, it transitions to `PASSAGES_RETRIEVED` — the retriever found the top-K passages and the card detail shows them with similarity scores.
4. Within ~10–30 s, the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWER_RECORDED`. The answer appears: a prose paragraph, a per-citation table (passage id, passage excerpt, claim it supports), and a `used_passages` count.
5. Within ~1 s of the answer, the `evalStep` finishes. The card shows a **groundedness score** chip (1–5) plus a one-line rationale describing whether the answer's claims are anchored in retrieved passages.
6. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → retrieved → answering → answer → eval. Source of truth. | `QueryEndpoint`, `PassageRetriever`, `QueryWorkflow` | `QueryView` |
| `PassageRetriever` | `Consumer` | Subscribes to `QuerySubmitted` events; runs cosine-similarity search over the in-process vector store; calls `QueryEntity.attachPassages`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitPassagesStep` → `answerStep` → `evalStep`. | started by `PassageRetriever` once passages are attached | `AnswerAgent`, `QueryEntity` |
| `AnswerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question in the task instructions and the retrieved passages as task attachments; returns `GroundedAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `GroundednessEvaluator` | — (plain class) | Deterministic scorer: checks that every `Citation.passageId` in the answer is present in the retrieved set, that citations are non-empty, and that the answer prose is not longer than the retrieved content combined. | invoked by `QueryWorkflow.evalStep` | returns `EvalResult` |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Passage(
    String passageId,
    String documentTitle,
    String text,
    double similarityScore   // cosine similarity, 0..1
) {}

record QueryRequest(
    String queryId,
    String questionText,
    String submittedBy,
    Instant submittedAt
) {}

record RetrievedPassages(
    List<Passage> passages,
    int topK
) {}

record Citation(
    String passageId,
    String passageExcerpt,   // verbatim excerpt from the Passage.text
    String claimSupported    // the part of the answer this passage anchors
) {}

record GroundedAnswer(
    String answerText,
    List<Citation> citations,
    int usedPassages,
    Instant answeredAt
) {}

record EvalResult(
    int score,               // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<RetrievedPassages> retrieved,
    Optional<GroundedAnswer> answer,
    Optional<EvalResult> eval,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, PASSAGES_RETRIEVED, ANSWERING, ANSWER_RECORDED, EVALUATED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `PassagesRetrieved`, `AnsweringStarted`, `AnswerRecorded`, `EvaluationScored`, `QueryFailed`.

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

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: RAG Sample</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted queries (status pill + groundedness chip + age) and a right pane with the selected query's detail — question text, retrieved passages with similarity scores, answer prose, citation table, and groundedness score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision groundedness eval** (`eval-event`, `on-decision-eval`): runs immediately after `AnswerRecorded` lands, as `evalStep` inside the workflow. A deterministic scorer (`GroundednessEvaluator` — no LLM call, so the single-agent invariant stays honest) checks: (1) every `Citation.passageId` exists in `retrieved.passages`; (2) no citation passageId is repeated; (3) `usedPassages` matches the actual citation count; (4) citations are non-empty. Score 5 = all checks pass with ≥ 3 citations; score 4 = all checks pass with 1–2 citations; score 3 = minor inconsistencies (usedPassages mismatch); score 1–2 = fabricated passage ids or empty citations. Emits `EvaluationScored` with a 1–5 score and a one-line rationale.

## 9. Agent prompts

- `AnswerAgent` → `prompts/answer-agent.md`. The single decision-making LLM. System prompt instructs it to read the retrieved passages (provided as task attachments), answer the question using only the passage content, and return a `GroundedAnswer` where every citation id maps to an attached passage.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seeded question; within 30 s the answer appears with at least one citation and a groundedness score chip.
2. **J2** — User submits a question for which the vector store returns zero relevant passages (similarity below threshold); the agent returns a `GroundedAnswer` with `citations` empty and `answerText` stating it found no relevant material; the eval scores it 1 and the card border highlights red.
3. **J3** — An answer where every citation id matches a retrieved passage scores 4 or 5; the UI shows the score chip in green.
4. **J4** — The SSE stream delivers one event per state transition; a browser that opens the stream after `ANSWERING` still receives `ANSWER_RECORDED` and `EVALUATED` events and reconciles the card correctly.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named rag-baseline demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-rag-baseline. Java package io.akka.samples.ragsample. Akka 3.6.0.
HTTP port 9846.

Components to wire (exactly):

- 1 AutonomousAgent AnswerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/answer-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUESTION).maxIterationsPerTask(2)). The task
  receives the question in its instruction text and each retrieved passage as a separate
  task ATTACHMENT named "passage-<passageId>.txt" (NOT as inline prompt text —
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  GroundedAnswer{answerText: String, citations: List<Citation>, usedPassages: int,
  answeredAt: Instant}.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitPassagesStep — polls QueryEntity.getQuery every 1s; on
    query.retrieved().isPresent() advances to answerStep.
    WorkflowSettings.stepTimeout 15s.
  * answerStep — emits AnsweringStarted, then calls
    componentClient.forAutonomousAgent(AnswerAgent.class, "answerer-" + queryId)
    .runSingleTask(
      TaskDef.instructions("Question: " + query.request().get().questionText())
        .attachment("passage-<id>.txt", passage.text().getBytes())
        // one .attachment() call per retrieved passage
    ) — returns a taskId, then forTask(taskId).result(ANSWER_QUESTION) to fetch the
    answer. On success calls QueryEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * evalStep — runs GroundednessEvaluator (NOT an LLM call) over the recorded answer
    and the retrieved passages: checks passage id membership, citation completeness, and
    usedPassages consistency. Emits EvaluationScored{score: 1-5, rationale: String}.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, retrieved: Optional<RetrievedPassages>,
  answer: Optional<GroundedAnswer>, eval: Optional<EvalResult>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED,
  PASSAGES_RETRIEVED, ANSWERING, ANSWER_RECORDED, EVALUATED, FAILED. Events:
  QuerySubmitted{request}, PassagesRetrieved{retrieved}, AnsweringStarted{},
  AnswerRecorded{answer}, EvaluationScored{eval}, QueryFailed{reason}. Commands:
  submit, attachPassages, markAnswering, recordAnswer, recordEvaluation, fail, getQuery.
  emptyState() returns Query.initial("") with all Optional fields as Optional.empty()
  and status = SUBMITTED. emptyState() never references commandContext() (Lesson 3).

- 1 Consumer PassageRetriever subscribed to QueryEntity events; on QuerySubmitted runs
  cosine-similarity search over an in-process VectorStore (loaded at startup from
  src/main/resources/corpus/passages.jsonl). Retrieves top-5 passages with similarity
  ≥ 0.0 (always returns something — the evaluator handles the zero-evidence case by
  scoring low). Builds RetrievedPassages{passages: List<Passage>, topK: 5}, then calls
  QueryEntity.attachPassages(retrieved). After attachPassages lands, the same Consumer
  starts a QueryWorkflow with id = "query-" + queryId.

- 1 View QueryView with row type QueryRow (mirrors Query). Table updater consumes
  QueryEntity events. ONE query getAllQueries: SELECT * AS queries FROM query_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {questionText, submittedBy};
    mints queryId; calls QueryEntity.submit; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task
  .name("Answer question")
  .description("Read the attached passages and produce a GroundedAnswer")
  .resultConformsTo(GroundedAnswer.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records Passage, QueryRequest, RetrievedPassages, Citation, GroundedAnswer,
  EvalResult, Query, QueryStatus.

- VectorStore.java — plain Java class loaded at startup from
  src/main/resources/corpus/passages.jsonl. Represents each passage as a float[]
  embedding (TF-IDF approximation over word overlap — no external model call, no HTTP).
  Exposes List<Passage> search(String query, int topK). This is intentionally
  rudimentary — the blueprint's point is the Akka orchestration, not embedding quality.

- GroundednessEvaluator.java — pure deterministic logic (no LLM). Inputs:
  GroundedAnswer and RetrievedPassages. Outputs: EvalResult. Scoring rubric documented
  in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9846 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  The AnswerAgent.definition() binds the configured provider via
  .modelProvider("${akka.javasdk.agent.default}") or the per-agent override pattern
  from the akka-context docs.

- src/main/resources/corpus/passages.jsonl with 30 seeded passages across 3 topic
  groups: Akka concepts (10 passages), distributed systems theory (10 passages), and
  event sourcing patterns (10 passages). Each passage: {passageId, documentTitle, text}
  where text is 2–4 sentences. These are the only documents the vector store indexes.

- src/main/resources/sample-events/seed-questions.jsonl with 5 seeded questions:
  3 questions the corpus answers well (high groundedness), 1 question the corpus
  answers partially (medium groundedness), and 1 question the corpus cannot answer
  (forces citations: [], groundedness score 1).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with pre-filled general-domain fields;
  deployer-specific fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/answer-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: RAG Sample", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of query cards; right = selected-query detail with question text,
  retrieved passages list with similarity scores, answer prose, citation table, and
  groundedness score chip). Browser title exactly:
  <title>Akka Sample: RAG Sample</title>. No subtitle on the Overview tab.

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
    answer-question.json — 8 GroundedAnswer entries. Each entry has an answerText
    paragraph and a citations array. 5 entries have citation passageIds that all exist
    in the seeded corpus (high groundedness, score 4–5). 2 entries have 1–2 fabricated
    passageIds mixed with real ones (score 1–2 from the evaluator). 1 entry has an
    empty citations list (score 1, forces the "no evidence" card state). The mock
    should select a fabricated-id entry on the FIRST iteration of every 4th query
    (modulo seed) so J3 is reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. AnswerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (answerStep 60s, awaitPassagesStep 15s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: QueryTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(GroundedAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9846 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only
  the reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. The DOM contains exactly five <section class="tab-panel"> elements
  — Overview, Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (AnswerAgent). The
  on-decision evaluator is rule-based (GroundednessEvaluator.java) and does NOT make
  an LLM call — keeping the pattern's "one agent" promise honest.
- Passages are passed as task ATTACHMENTS (one attachment per passage), never inlined
  into the agent's instruction text. Verify the generated answerStep uses
  TaskDef.attachment(...) and not string interpolation into the instruction text.
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
