# SPEC — bm25-rag-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BM25RagAgent.
**One-line pitch:** A user asks a question; one AI agent retrieves the most relevant passages from a bundled technical corpus using BM25 lexical search, receives those passages as task attachments (never as inline prompt text), and returns a structured answer with cited passage ids.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `CorpusQueryAgent` (AutonomousAgent) carries the entire answer; the surrounding components only prepare its retrieval context and audit its output. One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates every candidate answer before it leaves the agent loop: the answer must be parseable as a `QueryAnswer`, every `citations[].passageId` must match a passage id that was actually delivered as an attachment in the same task, and `answerType` must be a value in the allowed enum. On any violation the guardrail rejects the response and the agent retries within its iteration budget.

The blueprint also includes a deterministic **grounding scorer** that runs immediately after each `AnswerRecorded` event, producing a 1–5 score without making a second LLM call — keeping the single-agent invariant intact.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** textarea (or picks one of four seeded example questions about transformer architecture, tokenizers, training loops, or model fine-tuning).
2. The user optionally adjusts the **Top-K** slider (default 5; range 1–10) to control how many BM25 passages feed the agent.
3. The user clicks **Ask**. The UI POSTs to `/api/queries` and receives a `queryId`.
4. The query card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `PASSAGES_ATTACHED` — the retrieved passage ids and their titles are visible in the card detail.
5. Within ~10–30 s the workflow's `answerStep` completes. The card transitions to `ANSWERING` then `ANSWER_RECORDED`. The answer appears: the `answerType` badge (`DIRECT` / `PARTIAL` / `NO_ANSWER`), the answer text, and a citation list (passage id → title → snippet).
6. Within ~1 s of the answer, the `scoringStep` finishes. The card shows a **grounding score** chip (1–5) plus a one-line rationale.
7. The user can submit another question; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — ask, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: submitted → passages attached → answering → answer → scored. Source of truth. | `QueryEndpoint`, `BM25Retriever`, `QueryWorkflow` | `QueryView` |
| `BM25Retriever` | `Consumer` | Subscribes to `QuerySubmitted` events; runs BM25 search over the in-process index; calls `QueryEntity.attachPassages`. | `QueryEntity` events | `QueryEntity` |
| `QueryWorkflow` | `Workflow` | One workflow per query. Steps: `awaitPassagesStep` → `answerStep` → `scoringStep`. | started by `BM25Retriever` once passages land | `CorpusQueryAgent`, `QueryEntity` |
| `CorpusQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the question as task instructions and retrieved passages as task attachments; returns `QueryAnswer`. | invoked by `QueryWorkflow` | returns answer |
| `CitationGuardrail` | supporting class | Registered on `CorpusQueryAgent` via the before-agent-response hook. Validates citation correctness. | agent loop | agent loop |
| `GroundingScorer` | supporting class | Deterministic rule-based scorer run inside `scoringStep`. No LLM call. | `QueryWorkflow` | `QueryEntity` |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record PassageRef(
    String passageId,
    String title,
    String snippet,   // first 200 chars of the passage body
    float bm25Score
) {}

record QueryRequest(
    String queryId,
    String questionText,
    int topK,
    String submittedBy,
    Instant submittedAt
) {}

record RetrievedPassages(
    List<PassageRef> passages,   // ordered by bm25Score desc
    int totalDocsScanned
) {}

record Citation(
    String passageId,
    String quotedFragment   // verbatim from the passage body
) {}

record QueryAnswer(
    AnswerType answerType,
    String answerText,
    List<Citation> citations,
    Instant decidedAt
) {}
enum AnswerType { DIRECT, PARTIAL, NO_ANSWER }

record GroundingResult(
    int score,          // 1..5
    String rationale,
    Instant scoredAt
) {}

record Query(
    String queryId,
    Optional<QueryRequest> request,
    Optional<RetrievedPassages> passages,
    Optional<QueryAnswer> answer,
    Optional<GroundingResult> grounding,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    SUBMITTED, PASSAGES_ATTACHED, ANSWERING, ANSWER_RECORDED, SCORED, FAILED
}
```

Events on `QueryEntity`: `QuerySubmitted`, `PassagesAttached`, `AnsweringStarted`, `AnswerRecorded`, `GroundingScored`, `QueryFailed`.

Every nullable lifecycle field on the `Query` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ questionText, topK, submittedBy }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: BM25 RAG Agent</title>`.

The App UI tab is a two-column layout: a left column with the question submission panel and the live list of query cards (status pill + answer type badge + grounding chip + age), and a right pane with the selected query's detail — retrieved passages list, answer text, citation table, and grounding score widget.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `CorpusQueryAgent`. Asserts the candidate response is well-formed `QueryAnswer` JSON, every `citations[].passageId` was present in the set of passages delivered as attachments in this task, and `answerType` is in `{DIRECT, PARTIAL, NO_ANSWER}`. On failure, returns a structured `citation-not-grounded` or `invalid-response` error to the agent loop so the task retries within its iteration budget.

The grounding scorer (`GroundingScorer`) is not a control in the eval matrix because it does not block or modify the answer — it is an observability signal. The single control is the guardrail.

## 9. Agent prompts

- `CorpusQueryAgent` → `prompts/corpus-query-agent.md`. The single decision-making LLM. System prompt instructs it to read the attached passage files, synthesize an answer, and return only `Citation` entries whose `passageId` matches one of the delivered attachments.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a seeded question; within 30 s the answer appears with at least one citation pointing to a real retrieved passage id, and a grounding score chip.
2. **J2** — The agent's first response on a query cites a passageId not in the retrieved set (mock LLM path) — the guardrail rejects it; the second iteration produces a grounded answer; the UI never displays the ungrounded response.
3. **J3** — A question with no relevant passages in the corpus receives an answer with `answerType = NO_ANSWER` and an empty citations list; the grounding score is 5 (correctly identified no evidence).
4. **J4** — A query where the agent's citations include a `quotedFragment` that does not appear verbatim in the corresponding passage receives a grounding score of 1–2; the card is flagged for review.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named bm25-rag-agent demonstrating the single-agent × research-intel cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-bm25-rag-agent. Java package io.akka.samples.ragbm25. Akka 3.6.0.
HTTP port 9617.

Components to wire (exactly):

- 1 AutonomousAgent CorpusQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/corpus-query-agent.md>) and
  .capability(TaskAcceptance.of(QueryTasks.ANSWER_QUESTION).maxIterationsPerTask(3)).
  The task receives the question text as its instruction text and each retrieved passage as a
  separate task ATTACHMENT named "passage-<passageId>.txt" containing the passage body
  (NOT as inline prompt text — Akka's TaskDef.attachment(name, contentBytes) is the canonical
  call). Output: QueryAnswer{answerType: AnswerType (DIRECT/PARTIAL/NO_ANSWER),
  answerText: String, citations: List<Citation>, decidedAt: Instant}. The agent is configured
  with a before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the
  agent's guardrail-configuration block. On guardrail rejection the agent loop retries the
  response within its 3-iteration budget.

- 1 Workflow QueryWorkflow per queryId with three steps:
  * awaitPassagesStep — polls QueryEntity.getQuery every 1s; on query.passages().isPresent()
    advances to answerStep. WorkflowSettings.stepTimeout 15s (BM25 is in-process and fast).
  * answerStep — emits AnsweringStarted, then calls componentClient.forAutonomousAgent(
    CorpusQueryAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions(query.request.questionText)
        .attachment("passage-" + p.passageId() + ".txt", p.body().getBytes())
        // one attachment per retrieved passage
    ) — returns a taskId, then forTask(taskId).result(QueryTasks.ANSWER_QUESTION) to fetch the
    answer. On success calls QueryEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(QueryWorkflow::error).
  * scoringStep — runs a deterministic rule-based GroundingScorer (NOT an LLM call) over the
    recorded answer and the retrieved passages: checks that every Citation.passageId is present
    in passages, that every Citation.quotedFragment appears verbatim in the corresponding
    passage body, and that a DIRECT answer has at least one citation. Emits
    GroundingScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity QueryEntity (one per queryId). State Query{queryId: String,
  request: Optional<QueryRequest>, passages: Optional<RetrievedPassages>,
  answer: Optional<QueryAnswer>, grounding: Optional<GroundingResult>, status: QueryStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum: SUBMITTED,
  PASSAGES_ATTACHED, ANSWERING, ANSWER_RECORDED, SCORED, FAILED. Events: QuerySubmitted{request},
  PassagesAttached{passages}, AnsweringStarted{}, AnswerRecorded{answer},
  GroundingScored{grounding}, QueryFailed{reason}. Commands: submit, attachPassages,
  markAnswering, recordAnswer, recordGrounding, fail, getQuery. emptyState() returns
  Query.initial("") with no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer BM25Retriever subscribed to QueryEntity events; on QuerySubmitted runs
  a BM25 search over the in-process CorpusIndex (loaded from
  src/main/resources/corpus/passages.jsonl at startup), retrieves the top-K passages
  (K = request.topK()), builds RetrievedPassages, then calls QueryEntity.attachPassages(passages).
  After attachPassages lands, the same Consumer starts a QueryWorkflow with id =
  "query-" + queryId. The CorpusIndex is a singleton Component (or a static loaded field)
  built once — not rebuilt per query.

- 1 View QueryView with row type QueryRow (mirrors Query minus any large raw body fields
  that belong only in the entity). Table updater consumes QueryEntity events. ONE query
  getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {questionText, topK, submittedBy};
    mints queryId; calls QueryEntity.submit; returns {queryId}), GET /queries (list from
    getAllQueries sorted newest-first), GET /queries/{id} (one row), GET /queries/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- QueryTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question").description("Read the attached passage documents and produce a QueryAnswer
  citing only the provided passages").resultConformsTo(QueryAnswer.class). DO NOT skip
  this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PassageRef, QueryRequest, RetrievedPassages, Citation, QueryAnswer,
  AnswerType, GroundingResult, Query, QueryStatus.

- CitationGuardrail.java implementing the before-agent-response hook. Reads the candidate
  QueryAnswer from the LLM response, checks (1) it is parseable as QueryAnswer,
  (2) every citations[].passageId is in the set of attachment names that were delivered
  in this task (derived from the RetrievedPassages recorded on the entity),
  (3) answerType is in {DIRECT, PARTIAL, NO_ANSWER}. On any failure returns
  Guardrail.reject(<structured-error>).

- GroundingScorer.java — pure deterministic logic (no LLM). Inputs: QueryAnswer,
  RetrievedPassages (the full passage bodies for verbatim checking). Outputs: GroundingResult.
  Scoring rubric:
    +1 if answerType is NO_ANSWER and citations list is empty (correct restraint).
    +1 per citation whose quotedFragment appears verbatim in the corresponding passage body
       (up to 3 points from this check).
    +1 if answerType is DIRECT and at least one DIRECT citation is present.
    -1 if any Citation.passageId is not in the passage set (should not happen post-guardrail,
       but scored defensively).
  Final score clamped to [1, 5]. Rationale string names which checks passed/failed.

- CorpusIndex.java — in-process BM25 index. Loaded from
  src/main/resources/corpus/passages.jsonl at startup. Implements a standard
  Okapi BM25 (k1=1.5, b=0.75) over tokenized passage bodies. Exposes
  search(String query, int topK): List<PassageRef>. The index is built once
  and shared as an application-scoped singleton.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9617 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The CorpusQueryAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/corpus/passages.jsonl — a bundled corpus of 80 passages drawn from
  publicly available transformer documentation. Each entry: { "passageId": "p-<n>",
  "title": "<section heading>", "body": "<200-600 word passage>", "url": "<source url>" }.
  Covers: attention mechanisms, positional encoding, tokenization algorithms (BPE,
  WordPiece), training objectives (MLM, CLM, prefix-LM), fine-tuning strategies
  (LoRA, adapter layers, full fine-tune), and inference optimization (KV cache, quantization,
  batching). Each passage is self-contained and citable.

- src/main/resources/sample-events/seed-questions.jsonl — 4 seeded questions with their
  expected top-3 passage ids (for UI "Load example" functionality):
    1. "How does scaled dot-product attention work?"
    2. "What is the difference between BPE and WordPiece tokenization?"
    3. "How does LoRA reduce the number of trainable parameters?"
    4. "What is KV-cache and why does it speed up autoregressive decoding?"

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (the corpus
  is technical documentation with no PII), decisions.authority_level = informational
  (the agent surfaces references; the user decides what to trust), oversight.human_in_loop
  = false (the answer is advisory; no gated approval is needed for reference lookups),
  failure.failure_modes including "hallucinated-citation", "out-of-corpus-claim",
  "no-answer-when-relevant-passage-exists", "grounding-score-overfit";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/corpus-query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: RAG (BM25)", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  question submission panel + live list of query cards; right = selected-query detail with
  retrieved passages, answer text, citation table, grounding score widget).
  Browser title exactly: <title>Akka Sample: BM25 RAG Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; passed to
        the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads
  src/main/resources/mock-responses/<task-id>.json, picks one entry pseudo-randomly per
  call (seedFor(queryId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 8 QueryAnswer entries covering all three AnswerType values.
      Each DIRECT/PARTIAL entry has 1–3 Citation entries whose passageIds exist in the
      bundled corpus. Each Citation has a non-empty quotedFragment that is a verbatim
      excerpt from the corresponding passage. Plus 2 deliberately MALFORMED entries:
      one citing a passageId that does not exist in the corpus (guardrail blocks it);
      one with an answerType value outside the enum (guardrail blocks it). The mock
      selects a malformed entry on the FIRST iteration of every 3rd query (modulo seed)
      so J2 is reproducible.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. CorpusQueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (answerStep 60s,
  awaitPassagesStep 15s, scoringStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Query row record is Optional<T>.
- Lesson 7: QueryTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(QueryAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated models.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9617 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (CorpusQueryAgent).
  The grounding scorer is rule-based and does NOT make an LLM call.
- Passages are delivered as task ATTACHMENTS, never inlined into the agent's instruction
  text. Verify the generated answerStep uses TaskDef.attachment(...) per retrieved passage.
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
