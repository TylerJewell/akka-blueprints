# SPEC — sub-question-supervisor

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Sub-Question Query Engine.
**One-line pitch:** Ask a complex question; a supervisor decomposes it into sub-questions, dispatches each to an index tool in parallel, then synthesises a single combined answer.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans sub-questions out to multiple IndexWorker AutonomousAgent instances in parallel, collects their individual answers, and asks a QuerySupervisor AutonomousAgent to synthesise a final combined response. The blueprint also demonstrates a **before-tool-call guardrail** that vets each sub-question before it is dispatched to an index, and an **eval-event** that samples the supervisor's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a complex question via the form.

1. The system creates a `QuerySession` record in `DECOMPOSING` and starts a `QueryOrchestrationWorkflow`.
2. The `QuerySupervisor` breaks the question into `2–4` sub-questions, each tagged with a target index identifier (`index_id`).
3. The workflow forks: one `IndexWorker` per sub-question runs concurrently. Before each worker dispatches its index call, a before-tool-call guardrail inspects the sub-question for safety and relevance; an invalid sub-question causes that worker to short-circuit with an `IndexCallRejected` event.
4. Each `IndexWorker` returns an `IndexResult { subQuestion, indexId, answer, sourceRefs }`.
5. The workflow joins all results and asks the `QuerySupervisor` to synthesise them into a `CombinedAnswer { summary, indexResults, synthesisedAt }`.
6. The session moves to `SYNTHESISED`; the combined answer is surfaced in the UI.
7. If any worker times out after 60 seconds, the workflow short-circuits to a `partialStep`: the supervisor synthesises from whichever results arrived, and the session enters `PARTIAL`.

A `QuestionSimulator` (TimedAction) drips a sample question every 60 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QuerySupervisor` | `AutonomousAgent` | Decomposes the question into sub-questions; synthesises the collected index results into a combined answer. | `QueryOrchestrationWorkflow` | returns typed result to workflow |
| `IndexWorker` | `AutonomousAgent` | Retrieves a result from a designated index for one sub-question. Seeded "index tool" returns canned results per index_id. | `QueryOrchestrationWorkflow` | — |
| `QueryOrchestrationWorkflow` | `Workflow` | Coordinates parallel sub-question dispatch, the before-tool-call guardrail, and the synthesis step. | `QueryEndpoint`, `IndexCallConsumer` | `QuerySessionEntity` |
| `QuerySessionEntity` | `EventSourcedEntity` | Holds the session's lifecycle (decomposing → retrieving → synthesised / partial / blocked). | `QueryOrchestrationWorkflow` | `QuerySessionView` |
| `IndexCallQueue` | `EventSourcedEntity` | Logs each submitted question for replay and audit. | `QueryEndpoint`, `QuestionSimulator` | `IndexCallConsumer` |
| `QuerySessionView` | `View` | List-of-sessions read model. | `QuerySessionEntity` events | `QueryEndpoint` |
| `IndexCallConsumer` | `Consumer` | Listens to `IndexCallQueue` events and starts a workflow per submission. | `IndexCallQueue` events | `QueryOrchestrationWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample question every 60 s. | scheduler | `IndexCallQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesised session every 5 minutes for eval scoring; emits a `SynthesisEvalScored` event. | scheduler | `QuerySessionEntity` |
| `QueryEndpoint` | `HttpEndpoint` | `/api/query/*` — submit, get, list, SSE. | — | `QuerySessionView`, `IndexCallQueue`, `QuerySessionEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record QuestionRequest(String question, String requestedBy) {}

record SubQuestion(String text, String indexId) {}
record DecompositionPlan(List<SubQuestion> subQuestions, Instant decomposedAt) {}

record SourceRef(String title, String uri) {}
record IndexResult(String subQuestion, String indexId, String answer, List<SourceRef> sourceRefs, Instant retrievedAt) {}

record CombinedAnswer(String summary, List<IndexResult> indexResults, Instant synthesisedAt) {}

record QuerySession(
    String sessionId,
    String question,
    SessionStatus status,
    Optional<DecompositionPlan> decomposition,
    Optional<List<IndexResult>> indexResults,
    Optional<CombinedAnswer> combinedAnswer,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus { DECOMPOSING, RETRIEVING, SYNTHESISED, PARTIAL, BLOCKED }
```

### Events (on `QuerySessionEntity`)

`SessionCreated`, `DecompositionReady`, `IndexResultAttached`, `IndexCallRejected`, `SessionSynthesised`, `SessionPartial`, `SessionBlocked`, `SynthesisEvalScored`.

### Events (on `IndexCallQueue`)

`QuestionSubmitted { sessionId, question, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/query` — body `{ question }` → `{ sessionId }`. Starts a workflow.
- `GET /api/query` — list all sessions. Optional `?status=DECOMPOSING|RETRIEVING|SYNTHESISED|PARTIAL|BLOCKED`.
- `GET /api/query/{id}` — one session.
- `GET /api/query/sse` — server-sent events stream of every session change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Sub-Question Query Engine"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a question, live list of sessions with status pills, expand-row to see sub-questions, index results, combined summary, and eval score.

Browser title: `<title>Akka Sample: Sub-Question Query Engine</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — before-tool-call guardrail** (`before-tool-call` on `IndexWorker`): inspects each sub-question before the index call is dispatched. Blocks malformed, off-topic, or policy-violating sub-questions. Failure → `IndexCallRejected` event; if all sub-questions are rejected, the session enters `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one synthesised session every 5 minutes and emits a `SynthesisEvalScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `QuerySupervisor` → `prompts/supervisor.md`. Decomposes the question into sub-questions with index tags; later synthesises all index results into a combined answer.
- `IndexWorker` → `prompts/index-worker.md`. Retrieves a result for one sub-question from its designated index; returns `IndexResult`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a multi-part question; session progresses DECOMPOSING → RETRIEVING → SYNTHESISED within 90 s; UI reflects each transition via SSE.
2. **J2** — A sub-question is malformed (test fixture); the before-tool-call guardrail blocks that sub-question; the remaining sub-questions complete; session ends SYNTHESISED from partial results.
3. **J3** — All sub-questions are rejected; session enters BLOCKED with a `failureReason`.
4. **J4** — Inject a worker timeout (set `IndexWorker` step timeout to 1 s); session enters PARTIAL with whichever results arrived.
5. **J5** — Wait after a successful synthesis; the session's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named sub-question-supervisor demonstrating the
delegation-supervisor-workers × research-intel cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-research-intel-sub-question-supervisor.
Java package io.akka.samples.subquestionqueryengineworkflow. Akka 3.6.0. HTTP port 9985.

Components to wire (exactly):
- 2 AutonomousAgents:
  * QuerySupervisor — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/supervisor.md. Returns DecompositionPlan{subQuestions: List<SubQuestion{text, indexId}>,
    decomposedAt} for DECOMPOSE and CombinedAnswer{summary, indexResults: List<IndexResult>,
    synthesisedAt} for SYNTHESISE.
  * IndexWorker — capability(TaskAcceptance.of(RETRIEVE).maxIterationsPerTask(3)). System prompt from
    prompts/index-worker.md. Returns IndexResult{subQuestion, indexId, answer, sourceRefs:
    List<SourceRef{title, uri}>, retrievedAt}. Before the worker dispatches its seeded index
    call, a before-tool-call guardrail validates the sub-question for relevance and safety;
    invalid sub-questions throw a GuardrailException instead of calling the index tool.

- 1 Workflow QueryOrchestrationWorkflow with steps:
  decomposeStep -> [parallel] retrieveSteps -> joinStep -> synthesiseStep -> emitStep.
  decomposeStep calls forAutonomousAgent(QuerySupervisor.class, DECOMPOSE) with the question.
  retrieveSteps: for each SubQuestion in the DecompositionPlan, spawn one concurrent
  CompletionStage calling forAutonomousAgent(IndexWorker.class, RETRIEVE); zip all into a
  List<IndexResult>. Each retrieval step gets WorkflowSettings.builder().stepTimeout of
  ofSeconds(60). On any timeout, transition to partialStep that synthesises from whichever
  results arrived, then ends with SessionPartial. On GuardrailException from IndexWorker,
  record an IndexCallRejected event and continue with remaining sub-questions; if all are
  rejected, end with SessionBlocked. synthesiseStep calls forAutonomousAgent(QuerySupervisor.class,
  SYNTHESISE) with the collected IndexResults; give synthesiseStep a 90s stepTimeout.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity QuerySessionEntity holding state QuerySession{sessionId, question,
  SessionStatus, Optional<DecompositionPlan> decomposition, Optional<List<IndexResult>>
  indexResults, Optional<CombinedAnswer> combinedAnswer, Optional<String> failureReason,
  Optional<Integer> evalScore, Optional<String> evalRationale, Instant createdAt,
  Optional<Instant> finishedAt}. SessionStatus enum: DECOMPOSING, RETRIEVING, SYNTHESISED,
  PARTIAL, BLOCKED. Events: SessionCreated, DecompositionReady, IndexResultAttached,
  IndexCallRejected, SessionSynthesised, SessionPartial, SessionBlocked, SynthesisEvalScored.
  Commands: createSession, recordDecomposition, attachIndexResult, rejectIndexCall, synthesise,
  markPartial, block, recordEval, getSession. emptyState() returns QuerySession.initial("", null)
  with no commandContext() reference.

- 1 EventSourcedEntity IndexCallQueue with command enqueueQuestion(question, requestedBy)
  emitting QuestionSubmitted{sessionId, question, requestedBy, submittedAt}.

- 1 View QuerySessionView with row type QuerySessionRow (mirrors QuerySession minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  QuerySessionEntity events. ONE query getAllSessions SELECT * AS sessions FROM session_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer IndexCallConsumer subscribed to IndexCallQueue events; on QuestionSubmitted
  starts a QueryOrchestrationWorkflow with the sessionId as the workflow id.

- 2 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-questions.jsonl and calls
    IndexCallQueue.enqueueQuestion.
  * EvalSampler — every 5 minutes, queries QuerySessionView.getAllSessions, picks the oldest
    SYNTHESISED session without an evalScore, runs a 1–5 rubric judge over the combinedAnswer
    summary, then calls QuerySessionEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /query, GET /query, GET /query/{id},
    GET /query/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- QueryTasks.java declaring three Task<R> constants: DECOMPOSE (DecompositionPlan), RETRIEVE
  (IndexResult), SYNTHESISE (CombinedAnswer).
- Domain records SubQuestion, DecompositionPlan, SourceRef, IndexResult, CombinedAnswer,
  QuestionRequest.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9985 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-questions.jsonl with 8 canned multi-part question lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (H1 before-tool-call guardrail,
  E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = research and analysis,
  decisions.authority_level = recommend-only, data.pii = false, capabilities.* = false;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/supervisor.md, prompts/index-worker.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Sub-Question Query Engine", one-line
  pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Sub-Question Query Engine</title>. No
  subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (supervisor.json,
  index-worker.json), picks one entry pseudo-randomly per call, and
  deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    supervisor.json — list of either DecompositionPlan or CombinedAnswer objects.
      4–6 DecompositionPlan entries (each with 2–4 SubQuestion entries covering
      different indexId values such as "docs", "faq", "changelog", "api-ref") and
      4–6 CombinedAnswer entries (each with a 60–120 word summary and 2–4
      IndexResult entries referencing believable sourceRefs).
    index-worker.json — 4–6 IndexResult entries per seeded index_id, each with a
      2–4 sentence answer and 1–3 SourceRef entries whose uri values are plausible
      (e.g., "https://docs.example.com/guides/auth", "https://example.com/faq#billing").
- A MockModelProvider.seedFor(sessionId) helper makes selection deterministic
  per session id so the same session produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s retrieval
  workers, 90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion QueryTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9985 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
