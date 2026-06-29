# SPEC — multi-step-query-engine

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Multi-Step Query Engine.
**One-line pitch:** A user submits a research question; one `QueryAgent` walks it through three task phases — **DECOMPOSE** the question into sub-questions, **RETRIEVE** evidence passages for each sub-question, **SYNTHESIZE** a structured answer — with each phase gated on the prior phase's typed output and each emitted answer scored by a stopping-criterion evaluator.

## 2. What this blueprint demonstrates

The **sequential-pipeline** coordination pattern in a research-intelligence domain. One `QueryAgent` (AutonomousAgent) carries every LLM call across the pipeline. The pattern's defining property is the **explicit task dependency**: the DECOMPOSE task's typed output (a list of `SubQuestion` records) becomes the RETRIEVE task's instruction context; the RETRIEVE task's typed output (an `EvidenceSet`) becomes the SYNTHESIZE task's instruction context. The agent never holds all three phases in one conversation — each phase runs as its own typed task with its own scoped tool set.

One governance mechanism is wired around the pipeline:

- An **`on-decision-eval`** runs immediately after `AnswerSynthesized` lands, as `evalStep` inside the workflow. A deterministic, rule-based `StoppingEvaluator` (no LLM call — the eval is rule-based on purpose, so the same answer always scores the same) checks that every sub-question has at least one corresponding passage (`SubQuestion` coverage), every passage cited in the final answer actually appears in the retrieved `EvidenceSet` (no hallucinated citations), the answer's stated confidence aligns with the evidence density (at least one passage per sub-question required for full confidence), and the `QueryAnswer.sections` count equals the `SubQuestion` count (no silent expansion or collapse). The evaluator's score tells the reader when a result is evidence-thin and deserves additional retrieval steps or human verification.

The blueprint shows that a sequential pipeline is not just a chain of LLM calls — the task-boundary handoffs are the right cut to enforce the dependency contract and make the stopping criterion measurable after every synthesize run.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a **question** into the input (or picks one of three seeded questions — `What are the primary drivers of transformer model scaling laws?`, `How does retrieval-augmented generation compare to fine-tuning for domain adaptation?`, `What evidence exists for emergent capabilities in large language models?`).
2. The user clicks **Run query**. The UI POSTs to `/api/queries` and receives a `queryId`.
3. The card appears in the live list in `CREATED` state. Within ~1 s it transitions to `DECOMPOSING` — the workflow has started `decomposeStep` and the agent has been handed the DECOMPOSE task.
4. Within ~10–20 s the card reaches `RETRIEVING` — the typed list of `SubQuestion` records is visible in the card detail. The agent's DECOMPOSE task returned; the workflow recorded `QuestionDecomposed` and ran the RETRIEVE task.
5. Within ~10–20 s more the card reaches `SYNTHESIZING`. The `EvidenceSet` is visible (passage list with sub-question assignment, source, and relevance snippet).
6. Within ~10–20 s more the card reaches `EVALUATED`. The right pane now shows the full typed `QueryAnswer` — question, confidence level, per-section answer paragraphs with cited passages — plus an eval score chip (1–5) and a one-line rationale.
7. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `QueryEntity`, `QueryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `QueryEntity` | `EventSourcedEntity` | Per-query lifecycle: created → decomposing → decomposed → retrieving → retrieved → synthesizing → synthesized → evaluated. Source of truth. | `QueryEndpoint`, `QueryPipelineWorkflow` | `QueryView` |
| `QueryPipelineWorkflow` | `Workflow` | One workflow per query. Steps: `decomposeStep` → `retrieveStep` → `synthesizeStep` → `evalStep`. Each step runs one task on the agent, reads the typed result, writes the corresponding event onto the entity, then advances. | started by `QueryEndpoint` after `CREATED` | `QueryAgent`, `QueryEntity` |
| `QueryAgent` | `AutonomousAgent` | The single agent. Declares three `Task<R>` constants in `QueryTasks.java`: `DECOMPOSE_QUESTION` → `DecomposedQuestion`, `RETRIEVE_EVIDENCE` → `EvidenceSet`, `SYNTHESIZE_ANSWER` → `QueryAnswer`. Each task is registered with the phase-appropriate function tools. | invoked by `QueryPipelineWorkflow` | returns typed results |
| `DecomposeTools` | function-tools class (POJO with `@FunctionTool` methods) | Implements `generateSubQuestions(question)` and `rankSubQuestions(subQuestions)`. Reads from `src/main/resources/sample-data/questions/*.json` for deterministic offline output. | called from DECOMPOSE task | returns `List<SubQuestion>` |
| `RetrieveTools` | function-tools class | Implements `searchPassages(subQuestion)` and `fetchPassage(passageId)`. Reads from `src/main/resources/sample-data/passages/*.json`. | called from RETRIEVE task | returns `List<Passage>` |
| `SynthesizeTools` | function-tools class | Implements `draftSection(subQuestion, passages)` and `composeAnswer(sections)`. Pure in-memory transformations. | called from SYNTHESIZE task | returns `Section` / `QueryAnswer` |
| `StoppingEvaluator` | plain class (no Akka primitive) | Pure deterministic on-decision evaluator. Inputs: `QueryAnswer`, `EvidenceSet`, `DecomposedQuestion`. Output: `EvalResult{score, rationale}`. | called from `evalStep` | returns score |
| `QueryView` | `View` | Read model: one row per query for the UI. | `QueryEntity` events | `QueryEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record SubQuestion(String subQuestionId, String text, int priority) {}

record DecomposedQuestion(List<SubQuestion> subQuestions, Instant decomposedAt) {}

record Passage(
    String passageId,
    String subQuestionId,
    String source,
    String url,
    String text,
    float relevanceScore
) {}

record EvidenceSet(List<Passage> passages, Instant retrievedAt) {}

record AnswerSection(
    String subQuestionId,
    String heading,
    String body,
    List<String> citedPassageIds
) {}

record QueryAnswer(
    String question,
    String summary,
    ConfidenceLevel confidence,
    List<AnswerSection> sections,
    Instant synthesizedAt
) {}

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record QueryRecord(
    String queryId,
    Optional<String> question,
    Optional<DecomposedQuestion> decomposition,
    Optional<EvidenceSet> evidence,
    Optional<QueryAnswer> answer,
    Optional<EvalResult> eval,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    CREATED, DECOMPOSING, DECOMPOSED, RETRIEVING, RETRIEVED,
    SYNTHESIZING, SYNTHESIZED, EVALUATED, FAILED
}

enum ConfidenceLevel {
    HIGH, MEDIUM, LOW, INSUFFICIENT
}
```

Events on `QueryEntity`: `QueryCreated`, `DecomposeStarted`, `QuestionDecomposed`, `RetrieveStarted`, `EvidenceRetrieved`, `SynthesizeStarted`, `AnswerSynthesized`, `AnswerEvaluated`, `QueryFailed`.

Every nullable lifecycle field on the `QueryRecord` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/queries` — body `{ question }` → `{ queryId }`.
- `GET /api/queries` — list all queries, newest-first.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Multi-Step Query Engine</title>`.

The App UI tab is a two-column layout: a left rail with the live list of queries (status pill + question preview + age) and a right pane with the selected query's detail — question, decomposed sub-questions, retrieved passages table, synthesized answer sections, eval score chip, and a confidence badge.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — `on-decision-eval`**: runs immediately after `AnswerSynthesized` lands, as `evalStep` inside the workflow. `StoppingEvaluator` is a deterministic rule-based scorer (no LLM call — keeping the single-agent pipeline invariant honest): every sub-question must have at least one passage in the `EvidenceSet` covering it (sub-question coverage), every cited `passageId` in the final answer's sections must appear in the retrieved `EvidenceSet` (no hallucinated citations), the stated `ConfidenceLevel` must match the evidence density (HIGH requires ≥ 2 passages per sub-question, MEDIUM requires ≥ 1, LOW or INSUFFICIENT require a matching explanation), and the `sections` count must equal the `subQuestions` count (section parity). Emits `AnswerEvaluated{score:1..5, rationale}` on a one-point-per-rule basis with a clear sentence pinpointing the largest gap.

## 9. Agent prompts

- `QueryAgent` → `prompts/query-agent.md`. The single decision-making LLM. The system prompt instructs it to recognise its current task by Task name, only call tools whose names match that phase's tool set, treat each task's typed input as the entire context for that phase, and return the task's typed output.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits the seeded question `What are the primary drivers of transformer model scaling laws?`; within 60 s the query reaches `EVALUATED` with ≥ 2 sub-questions, ≥ 2 passages per sub-question, ≥ 2 answer sections, and an eval score chip of ≥ 4.
2. **J2** — A query whose mock-LLM trajectory produces an answer section citing a passageId absent from the retrieved `EvidenceSet` is scored ≤ 2 with a rationale naming the hallucinated citation; the UI flags the card.
3. **J3** — Each task's instructions, attachments, and tool calls are visible in the per-query trace (logged at `INFO`); the DECOMPOSE task's log shows only DECOMPOSE-tool calls, the RETRIEVE task's log shows only RETRIEVE-tool calls, the SYNTHESIZE task's log shows only SYNTHESIZE-tool calls.
4. **J4** — A question with no matching passage file in the offline corpus returns an answer with `confidence = INSUFFICIENT` and `sections = []`; the pipeline completes without error.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-step-query-engine demonstrating the sequential-pipeline x
research-intel cell. Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact sequential-pipeline-research-intel-multi-step-query. Java package
io.akka.samples.multistepqueryengine. Akka 3.6.0. HTTP port 9201.

Components to wire (exactly):

- 1 AutonomousAgent QueryAgent — extends akka.javasdk.agent.autonomous.AutonomousAgent.
  definition() returns AgentDefinition with .instructions(<system prompt loaded from
  prompts/query-agent.md>) and three .capability(TaskAcceptance.of(TASK).maxIterationsPerTask(
  4)) entries — one per declared Task. Function tools are registered with .tools(...) — the
  DECOMPOSE, RETRIEVE, and SYNTHESIZE tool sets are ALL registered on the agent; phase ordering
  is enforced by the task-boundary handoff, NOT by conditional .tools(...) wiring. On task
  boundary the agent never holds prior-phase context — the workflow rebuilds the instruction
  context from the entity's recorded typed result.

- 1 Workflow QueryPipelineWorkflow per queryId with four steps:
  * decomposeStep — emits DecomposeStarted on the entity, then calls componentClient
    .forAutonomousAgent(QueryAgent.class, "agent-" + queryId).runSingleTask(
      TaskDef.instructions("Question: " + question + "\nPhase: DECOMPOSE\nGenerate 2-4 focused
      sub-questions that together cover the original question.")
        .metadata("queryId", queryId)
        .metadata("phase", "DECOMPOSE")
        .taskType(QueryTasks.DECOMPOSE_QUESTION)
    ). Reads forTask(taskId).result(DECOMPOSE_QUESTION) to get DecomposedQuestion. Writes
    QueryEntity.recordDecomposition(decomposedQuestion). WorkflowSettings.stepTimeout 60s.
  * retrieveStep — emits RetrieveStarted, then runSingleTask with TaskDef.instructions
    (formatRetrieveContext(decomposedQuestion, question)) and metadata.phase = "RETRIEVE",
    taskType RETRIEVE_EVIDENCE. Writes QueryEntity.recordEvidence(evidenceSet). stepTimeout 60s.
  * synthesizeStep — emits SynthesizeStarted, then runSingleTask with TaskDef.instructions
    (formatSynthesizeContext(evidenceSet, decomposedQuestion, question)) and metadata.phase =
    "SYNTHESIZE", taskType SYNTHESIZE_ANSWER. Writes QueryEntity.recordAnswer(queryAnswer).
    stepTimeout 60s.
  * evalStep — runs the deterministic StoppingEvaluator over (queryAnswer, evidenceSet,
    decomposedQuestion) and writes QueryEntity.recordEvaluation(evalResult). stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5). settings() also declares defaultStepRecovery
  maxRetries(2).failoverTo(QueryPipelineWorkflow::error). The error step writes
  QueryFailed and ends.

- 1 EventSourcedEntity QueryEntity (one per queryId). State QueryRecord{queryId,
  question: Optional<String>, decomposition: Optional<DecomposedQuestion>,
  evidence: Optional<EvidenceSet>, answer: Optional<QueryAnswer>, eval: Optional<EvalResult>,
  status: QueryStatus, createdAt: Instant, finishedAt: Optional<Instant>}. QueryStatus enum:
  CREATED, DECOMPOSING, DECOMPOSED, RETRIEVING, RETRIEVED, SYNTHESIZING, SYNTHESIZED,
  EVALUATED, FAILED. Events: QueryCreated{question}, DecomposeStarted, QuestionDecomposed
  {decomposition}, RetrieveStarted, EvidenceRetrieved{evidence}, SynthesizeStarted,
  AnswerSynthesized{answer}, AnswerEvaluated{eval}, QueryFailed{reason}.
  Commands: create, startDecompose, recordDecomposition, startRetrieve, recordEvidence,
  startSynthesize, recordAnswer, recordEvaluation, fail, getQuery. emptyState() returns
  QueryRecord.initial("") with all Optional fields as Optional.empty() and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 View QueryView with row type QueryRow that mirrors QueryRecord exactly (all
  Optional<T> lifecycle fields preserved). Table updater consumes QueryEntity events. ONE
  query getAllQueries: SELECT * AS queries FROM query_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries (body {question}; mints queryId; calls
    QueryEntity.create(question); then starts QueryPipelineWorkflow with id
    "pipeline-" + queryId; returns {queryId}), GET /queries (list from
    getAllQueries, sorted newest-first), GET /queries/{id} (one row), GET
    /queries/sse (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion classes:

- QueryTasks.java declaring three Task<R> constants:
    DECOMPOSE_QUESTION = Task.name("Decompose question").description("Break the research
      question into 2-4 focused sub-questions").resultConformsTo(DecomposedQuestion.class);
    RETRIEVE_EVIDENCE = Task.name("Retrieve evidence").description("Find relevant passages
      for each sub-question using searchPassages and fetchPassage").resultConformsTo(
      EvidenceSet.class);
    SYNTHESIZE_ANSWER = Task.name("Synthesize answer").description("Compose a QueryAnswer
      whose sections mirror the sub-questions one-to-one").resultConformsTo(QueryAnswer.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- DecomposeTools.java — @FunctionTool generateSubQuestions(String question) -> List<SubQuestion>
  reading from src/main/resources/sample-data/questions/*.json keyed by question slug;
  @FunctionTool rankSubQuestions(List<SubQuestion>) -> List<SubQuestion> returning priority-
  sorted list.

- RetrieveTools.java — @FunctionTool searchPassages(String subQuestionId, String subQuestionText)
  -> List<Passage> reading from src/main/resources/sample-data/passages/*.json keyed by
  sub-question text; @FunctionTool fetchPassage(String passageId) -> Passage reading a single
  passage entry by id.

- SynthesizeTools.java — @FunctionTool draftSection(SubQuestion, List<Passage>) -> AnswerSection
  (heading from sub-question text, body composed from passage texts); @FunctionTool
  composeAnswer(String question, List<AnswerSection>, ConfidenceLevel) -> QueryAnswer (assembles
  the final answer record).

- StoppingEvaluator.java — pure deterministic logic (no LLM). Inputs: QueryAnswer,
  EvidenceSet, DecomposedQuestion. Outputs: EvalResult with score and rationale. Four checks,
  one point per check satisfied, starting from a base of 1:
  (1) sub-question coverage — every SubQuestion has at least one Passage in the EvidenceSet
  covering it (by matching subQuestionId);
  (2) citation provenance — every passageId cited in any AnswerSection.citedPassageIds appears
  in EvidenceSet.passages (no hallucinated citations);
  (3) confidence calibration — ConfidenceLevel HIGH requires ≥ 2 passages per sub-question,
  MEDIUM requires ≥ 1, LOW/INSUFFICIENT are self-consistent with any evidence count;
  (4) section parity — sections.size() == subQuestions.size() (no silent expansion or collapse).
  Score range 1-5. Rationale names the largest gap.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9201 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/questions.jsonl with 5 seeded question lines covering the
  three surfaces named in J1-J4 plus two extras.

- src/main/resources/sample-data/questions/*.json — three files keyed by question slug, each
  carrying 2-4 SubQuestion entries with deterministic text.

- src/main/resources/sample-data/passages/*.json — one file per question slug, each carrying
  6-10 Passage entries keyed by subQuestionId, with deterministic text and relevance scores.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (E1) matching the mechanism in Section 8
  of this SPEC. Matching simplified_view list. No regulation_anchors — research-intel domain.

- risk-survey.yaml at the project root with data.data_classes.pii = false (questions are
  topic-level, not person-level), decisions.authority_level = recommend-only (the answer is
  advisory), oversight.human_in_loop = true (a researcher reads the answer before acting on
  it), operations.agent_count = 1, operations.agent_pattern = sequential-pipeline,
  failure.failure_modes including "hallucinated-citation", "insufficient-evidence",
  "missing-subquestion-coverage", "confidence-miscalibration"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Multi-Step Query Engine", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of query cards; right = selected-query detail with question header, sub-questions
  list, passages table, answer sections, eval-score chip, confidence badge).
  Browser title exactly: <title>Akka Sample: Multi-Step Query Engine</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns deterministic
        random-but-typed-correct outputs per Task (see Mock LLM provider block below). Sets
        model-provider = mock.
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(queryId)), and
  deserialises into the task's typed return. Within a single task run, the mock also drives
  tool-call sequences — each entry carries a "tool_calls" array the mock replays in order
  before returning the final typed result.
- Per-task mock-response shapes for THIS blueprint:
    decompose-question.json — 5 DecomposedQuestion entries, each with 2-4 SubQuestion items
      per seeded question. Each entry's tool_calls array contains 1-2 calls:
      generateSubQuestions(question) + optionally rankSubQuestions(subQuestions).
    retrieve-evidence.json — 5 EvidenceSet entries paired one-to-one with the decompose entries,
      each with 4-8 Passage items distributed across sub-questions. tool_calls contain
      searchPassages (one per sub-question) + 1-2 fetchPassage calls.
    synthesize-answer.json — 5 QueryAnswer entries paired one-to-one. Each carries AnswerSection
      items matching the sub-questions, citedPassageIds referencing the paired EvidenceSet,
      tool_calls containing draftSection (one per sub-question) + composeAnswer.
      Plus 1 deliberately HALLUCINATED-CITATION entry whose first AnswerSection cites a
      passageId absent from the paired EvidenceSet — the workflow's evalStep scores it ≤ 2;
      J2 verifies this.
- A MockModelProvider.seedFor(queryId) helper makes per-query selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. QueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion QueryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (decomposeStep
  60s, retrieveStep 60s, synthesizeStep 60s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the QueryRecord row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: QueryTasks.java with DECOMPOSE_QUESTION, RETRIEVE_EVIDENCE, SYNTHESIZE_ANSWER
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9201 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (QueryAgent). The
  on-decision eval is rule-based (StoppingEvaluator.java) and does NOT make an LLM call.
- The sequential-pipeline invariant: each phase's tool set is registered on the agent, but
  the task-boundary handoff is the runtime mechanism that enforces phase order. The agent
  never holds multi-phase context in a single conversation.
- Task dependency is carried by typed task results: decomposeStep writes DecomposedQuestion
  onto the entity, retrieveStep reads it and builds the RETRIEVE task's instruction context
  from it, synthesizeStep reads both. The agent itself is stateless across phases.
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
