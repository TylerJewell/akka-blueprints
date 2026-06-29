# SPEC — router-query-engine

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Router Query Engine Workflow.
**One-line pitch:** A router agent classifies an incoming research question and hands the retrieval task off to a structured-data engine or a semantic-search engine that owns the answer end-to-end, with an inline eval scoring every routing decision.

## 2. What this blueprint demonstrates

The **handoff-routing** coordination pattern — one classifier agent decides *which* retrieval engine should own the question, then transfers the same task identity to the chosen engine which produces the final answer. The downstream engine is responsible for the whole retrieval and synthesis; the router does not narrate or summarise.

One governance mechanism is layered on top:

- An **on-decision eval** fires every time the router agent emits a routing decision. A `RoutingJudge` agent grades the decision against the question on a 1–5 rubric. The score and rationale are written back to the query record and surfaced in the UI.

The pattern is a textbook fan-out-of-one: the workflow branches on the router's engine type, and only the chosen engine is invoked. The other engine sees no traffic for that question.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live query list. Every entry displays its engine-type chip, status pill, routing score, and (if answered) the published response.
2. `QuestionSimulator` (TimedAction) ticks every 30 s and inserts a new canned question from `sample-events/research-questions.jsonl` into `QuestionQueue`.
3. For each new question: `QuestionQueue` receives it; the `QueryWorkflow` is started directly from the `QuestionQueue` Consumer (`QuestionConsumer`).
4. The workflow calls `RouterAgent`, gets a `RoutingDecision { engineType, confidence, reason }`, and emits `RoutingDecided` on the query entity.
5. Branch on `engineType`:
   - `STRUCTURED` → workflow calls `StructuredDataEngine` with the `ANSWER` task and waits for the typed `Answer` result.
   - `SEMANTIC` → workflow calls `SemanticSearchEngine` with the same `ANSWER` task.
   - `UNCLEAR` → workflow emits `QueryEscalated`; ends.
6. The engine's `Answer` is published: `AnswerPublished` is emitted (terminal `ANSWERED`).
7. Independent of the workflow, `RoutingEvalScorer` (Consumer) listens for `RoutingDecided` events, calls `RoutingJudge`, and writes `RoutingScored { score, rationale }` back to the query entity.
8. The user can click any query card and see the original question, the routing decision, the routing score, the chosen engine's answer, and the published response.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QuestionSimulator` | `TimedAction` | Drips simulated research questions into `QuestionQueue` every 30 s. | scheduler | `QuestionQueue` |
| `QuestionQueue` | `EventSourcedEntity` | Append-only audit log of every inbound question (`InboundQuestionReceived`). | `QuestionSimulator`, `QueryEndpoint` | `QuestionConsumer` |
| `QuestionConsumer` | `Consumer` | Subscribes to `QuestionQueue` events; registers `QueryEntity`; starts a `QueryWorkflow`. | `QuestionQueue` events | `QueryEntity`, `QueryWorkflow` |
| `RouterAgent` | `Agent` (typed, not autonomous) | Classifies a `ResearchQuestion` into `STRUCTURED` / `SEMANTIC` / `UNCLEAR` with confidence + reason. | invoked by `QueryWorkflow` | returns `RoutingDecision` |
| `StructuredDataEngine` | `AutonomousAgent` | Owns the `ANSWER` task for structured-data questions (aggregations, filters, precise lookups). Returns typed `Answer`. | invoked by `QueryWorkflow` | returns `Answer` |
| `SemanticSearchEngine` | `AutonomousAgent` | Owns the `ANSWER` task for semantic questions (concept search, summarization, open-ended synthesis). Returns typed `Answer`. | invoked by `QueryWorkflow` | returns `Answer` |
| `RoutingJudge` | `Agent` (typed) | Grades a routing decision against the original question. Returns `RoutingScore { score 1–5, rationale }`. | invoked by `RoutingEvalScorer` | returns `RoutingScore` |
| `QueryWorkflow` | `Workflow` | Per-question orchestration: route → branch → answer → publish. | `QuestionConsumer` (start) | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Per-question lifecycle. | `QueryWorkflow`, `RoutingEvalScorer` | `QueryView` |
| `QueryView` | `View` | Read-model row per query. | `QueryEntity` events | `QueryEndpoint` |
| `RoutingEvalScorer` | `Consumer` | Subscribes to `QueryEntity` events; on `RoutingDecided` invokes `RoutingJudge` and writes `RoutingScored` back. | `QueryEntity` events | `QueryEntity` |
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — list, get, manual submit, SSE; `/api/metadata/*`. | — | `QueryView`, `QueryEntity`, `QuestionQueue` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record ResearchQuestion(
    String queryId,
    String questionText,
    String source,             // "simulator" | "manual"
    Instant receivedAt
) {}

enum EngineType { STRUCTURED, SEMANTIC, UNCLEAR }

record RoutingDecision(
    EngineType engineType,
    String confidence,         // "high" | "medium" | "low"
    String reason              // one short sentence
) {}

enum AnswerAction { DIRECT_LOOKUP, AGGREGATION_RESULT, CONCEPT_SUMMARY, SYNTHESIS, ESCALATED }

record Answer(
    String answerText,
    AnswerAction action,
    String engineTag,          // "structured" | "semantic"
    List<String> sourceRefs,   // document ids or table names cited
    Instant answeredAt
) {}

record RoutingScore(
    int score,                 // 1..5
    String rationale,
    Instant scoredAt
) {}

record Query(
    String queryId,
    ResearchQuestion question,
    Optional<RoutingDecision> routing,
    Optional<Answer> answer,
    Optional<RoutingScore> routingScore,
    Optional<String> escalationReason,
    QueryStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus {
    RECEIVED,
    ROUTING,
    ROUTED_STRUCTURED,
    ROUTED_SEMANTIC,
    ANSWERING,
    ANSWERED,
    ESCALATED
}
```

Events on `QueryEntity`: `QueryRegistered`, `RoutingDecided`, `QueryRouted`, `AnswerDrafted`, `AnswerPublished`, `QueryEscalated`, `RoutingScored`.

Events on `QuestionQueue`: `InboundQuestionReceived`.

See `reference/data-model.md`.

## 6. API contract

- `GET /api/queries` — list all queries (newest-first), optional `?engineType=STRUCTURED|SEMANTIC|UNCLEAR&status=…` filtered client-side.
- `GET /api/queries/{id}` — one query.
- `POST /api/queries` — manually submit a question (body `ResearchQuestion` minus `queryId` and `receivedAt`); server assigns both.
- `GET /api/queries/sse` — Server-Sent Events for every query change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Router Query Engine Workflow</title>`.

The App UI tab is a two-pane layout: **left** is the query list (status pill + engine-type chip + routing score chip), **right** is the selected query's question text, routing decision block, routing score block, and the chosen engine's answer with source references.

Tab switching is attribute-based (`data-tab` / `data-panel`); no zombie panels in the DOM. The Architecture tab's mermaid diagrams carry the CSS overrides from Lesson 24 so state labels render white-on-dark and edge labels are not clipped.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — on-decision eval** (`eval-event`, on the routing decision): `RoutingEvalScorer` (Consumer) listens for `RoutingDecided` events and calls `RoutingJudge` to produce a 1–5 score with a one-sentence rationale. Non-blocking — the score is metadata, not a gate; persistent low scores would be surfaced as a system-level alert in a deployed setting.

## 9. Agent prompts

- `RouterAgent` → `prompts/router-agent.md`. Typed classifier; returns one of `STRUCTURED`, `SEMANTIC`, `UNCLEAR`; defaults to `UNCLEAR` under ambiguity.
- `StructuredDataEngine` → `prompts/structured-data-engine.md`. Owns the `ANSWER` task for structured-data questions. Never fabricates row counts or aggregation results; escalates when the question requires joins it cannot verify.
- `SemanticSearchEngine` → `prompts/semantic-search-engine.md`. Owns the `ANSWER` task for semantic questions. Cites document section ids when available; never invents citations.
- `RoutingJudge` → `prompts/routing-judge.md`. Grades a routing decision against a 1–5 rubric with a one-sentence rationale.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drips a structured-data question → routed `STRUCTURED` → answered by `StructuredDataEngine` → published.
2. **J2** — Simulator drips a semantic question → routed `SEMANTIC` → answered by `SemanticSearchEngine` → published.
3. **J3** — An ambiguous question routes to `UNCLEAR` and lands in `ESCALATED` without any engine invocation.
4. **J4** — Every routed question carries a `RoutingScore` (1–5) and rationale within ~10 s of the routing decision.
5. **J5** — A manually submitted question via `POST /api/queries` follows the same routing-and-answer path.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named router-query-engine demonstrating the handoff-routing × research-intel cell.
Runs out of the box (in-process simulated inbound question stream; no real corpus integration).
Maven group io.akka.samples. Maven artifact handoff-routing-research-intel-router-query-engine.
Java package io.akka.samples.routerqueryengineworkflow. Akka 3.6.0. HTTP port 9138.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) RouterAgent — classifier. System prompt loaded from
  prompts/router-agent.md. Input: ResearchQuestion{queryId, questionText, source,
  receivedAt}. Output: RoutingDecision{engineType: EngineType (STRUCTURED/SEMANTIC/UNCLEAR),
  confidence: "high"|"medium"|"low", reason: String}.
  Defaults to UNCLEAR under uncertainty.

- 1 AutonomousAgent StructuredDataEngine — definition() with
  capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(3)). System prompt from
  prompts/structured-data-engine.md. Input: ResearchQuestion + RoutingDecision. Output:
  Answer{answerText, action: AnswerAction, engineTag = "structured", sourceRefs: List<String>,
  answeredAt}. Never fabricates aggregation results; sets action=ESCALATED when the question
  requires data it cannot verify.

- 1 AutonomousAgent SemanticSearchEngine — definition() with
  capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(3)). System prompt from
  prompts/semantic-search-engine.md. Same input shape; engineTag = "semantic".

- 1 Agent (typed) RoutingJudge — judge. System prompt from prompts/routing-judge.md. Input:
  ResearchQuestion + RoutingDecision. Output: RoutingScore{score: int 1–5, rationale: String,
  scoredAt: Instant}.

- 1 Workflow QueryWorkflow per queryId. Steps:
    routeStep -> branchStep -> {structuredStep | semanticStep | escalateStep} -> publishStep
  routeStep calls componentClient.forAgent().inSession(queryId).method(RouterAgent::route)
    .invoke(question). On success emits RoutingDecided via QueryEntity.recordRouting.
  branchStep branches on RoutingDecision.engineType:
    STRUCTURED -> proceed to structuredStep (emits QueryRouted{STRUCTURED})
    SEMANTIC   -> proceed to semanticStep   (emits QueryRouted{SEMANTIC})
    UNCLEAR    -> escalateStep (emits QueryEscalated; terminates).
  structuredStep / semanticStep call forAutonomousAgent(<Engine>.class, queryId)
    .runSingleTask(TaskDef.instructions(buildPrompt(question, routing))) returning a taskId,
    then forTask(taskId).result(QueryTasks.ANSWER) to block on the typed Answer.
    On success emits AnswerDrafted.
  publishStep emits AnswerPublished (terminal ANSWERED).
  Override settings() with stepTimeout(Duration.ofSeconds(20)) on routeStep,
    stepTimeout(Duration.ofSeconds(60)) on structuredStep, semanticStep, and publishStep.
    defaultStepRecovery(maxRetries(2).failoverTo(QueryWorkflow::error)).

- 2 EventSourcedEntities:
    * QuestionQueue — append-only audit log. Command receive(ResearchQuestion) emits
      InboundQuestionReceived{question}. No mutable state beyond a counter; commands are
      idempotent on question.queryId.
    * QueryEntity (one per queryId) — full per-question lifecycle. State
      Query{queryId, question: ResearchQuestion, Optional<RoutingDecision> routing,
      Optional<Answer> answer, Optional<RoutingScore> routingScore,
      Optional<String> escalationReason, QueryStatus status, Instant createdAt,
      Optional<Instant> finishedAt}. QueryStatus enum: RECEIVED, ROUTING, ROUTED_STRUCTURED,
      ROUTED_SEMANTIC, ANSWERING, ANSWERED, ESCALATED.
      Events: QueryRegistered, RoutingDecided, QueryRouted, AnswerDrafted, AnswerPublished,
      QueryEscalated, RoutingScored. Commands: registerQuestion, recordRouting, recordRouted,
      recordAnswerDraft, publishAnswer, escalate, recordRoutingScore, getQuery.
      emptyState() returns Query.initial("") with no commandContext() reference.

- 2 Consumers:
    * QuestionConsumer subscribed to QuestionQueue events; for each InboundQuestionReceived
      calls QueryEntity.registerQuestion for the queryId, then starts a QueryWorkflow
      with queryId as the workflow id.
    * RoutingEvalScorer subscribed to QueryEntity events; on RoutingDecided invokes
      RoutingJudge.score(question, decision) and calls QueryEntity.recordRoutingScore(
      queryId, score). On any other event type, no-op. Use componentClient — do NOT
      call the agent from a TimedAction.

- 1 View QueryView with row type QueryRow (mirrors Query; uses Optional<T> for every
  nullable lifecycle field per Lesson 6). Table updater consumes QueryEntity events.
  ONE query getAllQueries SELECT * AS queries FROM query_view. No WHERE engineType or
  WHERE status filter (Akka cannot auto-index enum columns) — filter client-side in callers.

- 1 TimedAction QuestionSimulator — every 30s, reads next line from
  src/main/resources/sample-events/research-questions.jsonl (loops at EOF) and calls
  QuestionQueue.receive with a fresh queryId (UUID).

- 2 HttpEndpoints:
    * QueryEndpoint at /api with GET /queries (list from QueryView.getAllQueries,
      filter client-side by ?engineType and ?status query params), GET /queries/{id},
      POST /queries (body ResearchQuestion minus queryId/receivedAt — server assigns),
      GET /queries/sse (serverSentEventsForView over getAllQueries), and three
      /api/metadata/{readme,risk-survey,eval-matrix} endpoints serving the YAML/MD files
      from src/main/resources/metadata/.
    * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- QueryTasks.java declaring the task constant: ANSWER (resultConformsTo Answer.class,
  description "Retrieve and synthesize the answer to the research question end-to-end
  and return a typed Answer").
- Domain records ResearchQuestion, RoutingDecision, Answer, RoutingScore, and the Query
  entity state.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9138 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/research-questions.jsonl with 9 canned lines (3
  STRUCTURED-flavoured aggregation/lookup questions, 3 SEMANTIC-flavoured concept/synthesis
  questions, 2 UNCLEAR-flavoured ambiguous or off-topic questions, 1 specifically designed
  to route to STRUCTURED but require a join the engine must escalate on).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 1 control: E1 eval-event on-decision-eval.
  Matching simplified_view list. No regulation_anchors (community-content sample).
- risk-survey.yaml at the project root with purpose.primary_function = research-intelligence,
  data.data_classes.pii = false, decisions.authority_level = draft-only,
  oversight.human_in_loop = false (the system publishes without HITL — only escalated
  queries wait for a human), failure.failure_modes including "wrong-engine-routing",
  "fabricated-source-references", "incorrect-aggregation-result", "missed-relevant-documents";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router-agent.md, prompts/structured-data-engine.md, prompts/semantic-search-engine.md,
  prompts/routing-judge.md loaded as agent system prompts.
- README.md at the project root: title "Akka Sample: Router Query Engine Workflow",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = query list with engine-type chip + status pill + routing score chip;
  right = question text + routing decision + routing score + engine answer + source refs).
  Browser title exactly: <title>Akka Sample: Router Query Engine Workflow</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call (seeded by queryId so reruns are deterministic),
  and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    router-agent.json — 12 RoutingDecision entries spanning STRUCTURED (count
      queries, filter queries, precise lookups), SEMANTIC (concept search,
      trend summarization, comparative synthesis), and UNCLEAR (ambiguous
      one-liners, off-topic, mixed intent). Confidence + a one-sentence reason
      on each.
    structured-data-engine.json — 8 Answer entries: 5 with action
      DIRECT_LOOKUP or AGGREGATION_RESULT (within the engine's scope), 1 with
      action ESCALATED (join required beyond authority), 1 with CONCEPT_SUMMARY
      (edge case where structured data has narrative sections), 1 designed to
      expose a fabricated row count so routing-judge tests have material.
    semantic-search-engine.json — 8 Answer entries: 5 with CONCEPT_SUMMARY
      or SYNTHESIS, 1 with ESCALATED, 1 with a valid sourceRefs list, 1 with
      fabricated sourceRefs to stress-test the judge.
    routing-judge.json — 10 RoutingScore entries, score 1–5, one-sentence
      rationale matching the rubric (engine-type correctness / confidence-
      calibration / reason-quality).
- A MockModelProvider.seedFor(queryId) helper makes per-query selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent is never silently downgraded to Agent.
  StructuredDataEngine and SemanticSearchEngine both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and declare definition().
- (Lesson 4) Workflow step timeouts overridden via settings(): routeStep 20s,
  structuredStep / semanticStep / publishStep 60s each.
- (Lesson 6) Every nullable lifecycle field on Query is Optional<T>. The
  QueryView row type uses the same Optional wrapping.
- (Lesson 7) QueryTasks.java declares the ANSWER Task<Answer> constant.
  Both engines' definition().capability(TaskAcceptance.of(ANSWER)...)
  reference it.
- (Lesson 8) Model names verified against current lineup: claude-sonnet-4-6,
  gpt-4o, gemini-2.5-flash.
- (Lesson 9) Run command is "/akka:build" everywhere. No "mvn akka:run".
- (Lesson 10) Port 9138 in application.conf; not 9000.
- (Lesson 11) No source.platform string anywhere user-facing.
- (Lesson 12) Static UI fits in 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier label is "Runs out of the box" — never T1.
- (Lesson 23) No competitor brand names in any user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) API key sourcing follows the five-option protocol above.
- (Lesson 26) Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels.
- The RoutingEvalScorer Consumer reacts to RoutingDecided events and calls
  RoutingJudge via componentClient.forAgent(). It does NOT modify the workflow
  flow — the eval is out-of-band metadata.
- The ANSWER task is published after the engine returns — there is no
  separate guardrail step in this blueprint.
- No forbidden words in user-facing text: shape, minimal, smaller, complex,
  Akka SDK in narrative, T1/T2/T3/T4, deferred, use, use, marketing
  tone, competitor brand names.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the project-local key-source reference written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
