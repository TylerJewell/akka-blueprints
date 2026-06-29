# SPEC — multi-strategy-workflow

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Multi-Strategy Workflow.
**One-line pitch:** Submit a query; a coordinator briefs three independent strategy agents — Keyword Search, Semantic Retrieval, and Chain-of-Thought Reasoning — runs them concurrently over the same question, and synthesizes their answers into one authoritative response with per-strategy attribution.

## 2. What this blueprint demonstrates

The **debate-multi-perspective** coordination pattern wired with Akka's first-party primitives: a Workflow asks one coordinator agent to decompose a query into three strategy briefs, runs those strategy agents in parallel over the same input, then asks the coordinator to synthesize three independent answers into a single response. The blueprint also demonstrates two governance mechanisms — an **input guardrail** that blocks prohibited or malformed queries before any strategy is dispatched, and an **eval-event** sampler that scores how much the three strategy answers agree with the synthesized answer.

## 3. User-facing flows

The user opens the App UI tab and submits a query (question text) via the form.

1. The system creates a `Query` record in `RECEIVED` and starts a `QueryWorkflow`.
2. An input guardrail inspects the raw query for prohibited content and structural validity (non-empty, under 2000 characters). If it fails, the query moves to `REJECTED` and no strategy is dispatched.
3. The Coordinator decomposes the query into three strategy briefs: a keyword brief, a semantic brief, and a chain-of-thought brief.
4. The workflow forks: `KeywordSearchAgent`, `SemanticRetrievalAgent`, and `ChainOfThoughtAgent` run concurrently over the query. Each returns a typed `StrategyResult` with a per-strategy answer, a confidence score, and a list of supporting evidence items. The query moves to `RUNNING`.
5. The Coordinator synthesizes the three strategy results into a `SynthesizedAnswer { answer, summary, strategyResults, guardrailVerdict, synthesizedAt }`.
6. An output guardrail vets the synthesized answer; if it fails, the query moves to `BLOCKED`. Otherwise, the query moves to `SYNTHESIZED`.
7. If any strategy agent times out after 60 seconds, the workflow short-circuits: the Coordinator synthesizes from whichever strategies returned, and the query enters `DEGRADED`.
8. Every five minutes, an eval sampler picks one synthesized query and asks a judge whether the three strategy answers agree with the synthesized answer, attaching a 1–5 agreement score.

A `QuerySimulator` (TimedAction) drips a sample query every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StrategyCoordinator` | `AutonomousAgent` | Decomposes the query into three strategy briefs; later synthesizes the three strategy results into one answer. | `QueryWorkflow` | returns typed result to workflow |
| `KeywordSearchAgent` | `AutonomousAgent` | Executes keyword-based retrieval and ranks matching passages for the query. | `QueryWorkflow` | — |
| `SemanticRetrievalAgent` | `AutonomousAgent` | Performs vector-similarity reasoning over the query to retrieve the most semantically relevant answer. | `QueryWorkflow` | — |
| `ChainOfThoughtAgent` | `AutonomousAgent` | Generates a structured step-by-step reasoning chain and derives an answer from first principles. | `QueryWorkflow` | — |
| `ConsistencyJudge` | `AutonomousAgent` | Scores how much the three strategy answers agree with the synthesized answer. | `EvalSampler` | — |
| `QueryWorkflow` | `Workflow` | Coordinates the input guardrail, the parallel strategy fan-out, the synthesis, and the output guardrail. | `QueryEndpoint`, `QuerySimulatorConsumer` | `QueryEntity` |
| `QueryEntity` | `EventSourcedEntity` | Holds the query lifecycle (received → running → synthesized / degraded / blocked / rejected). | `QueryWorkflow`, `EvalSampler` | `QueryView` |
| `QueryQueue` | `EventSourcedEntity` | Logs each submitted query for replay/audit. | `QueryEndpoint`, `QuerySimulator` | `QuerySimulatorConsumer` |
| `QueryView` | `View` | List-of-queries read model. | `QueryEntity` events | `QueryEndpoint` |
| `QuerySimulatorConsumer` | `Consumer` | Listens to `QueryQueue` events and starts a workflow per submission. | `QueryQueue` events | `QueryWorkflow` |
| `QuerySimulator` | `TimedAction` | Drips a sample query every 60 s. | scheduler | `QueryQueue` |
| `EvalSampler` | `TimedAction` | Samples one synthesized query every 5 minutes for agreement scoring. | scheduler | `ConsistencyJudge`, `QueryEntity` |
| `QueryEndpoint` | `HttpEndpoint` | `/api/queries/*` — submit, get, list, SSE. | — | `QueryView`, `QueryQueue`, `QueryEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record QuerySubmission(String question, String submittedBy) {}

record StrategyBrief(String keywordBrief, String semanticBrief, String chainOfThoughtBrief) {}

record EvidenceItem(String source, String excerpt, double relevanceScore) {}
record StrategyResult(String strategy, String answer, double confidence,
                      List<EvidenceItem> evidence, Instant completedAt) {}
                      // strategy: KEYWORD | SEMANTIC | CHAIN_OF_THOUGHT
                      // confidence: 0.0–1.0

record SynthesizedAnswer(String answer, String summary,
                         List<StrategyResult> strategyResults,
                         String guardrailVerdict, Instant synthesizedAt) {}
                         // guardrailVerdict: "ok" or "blocked: <reason>"

record AgreementVerdict(int score, String rationale) {}  // score 1–5

record Query(
    String queryId,
    String question,
    QueryStatus status,
    Optional<StrategyResult> keywordResult,
    Optional<StrategyResult> semanticResult,
    Optional<StrategyResult> chainOfThoughtResult,
    Optional<SynthesizedAnswer> answer,
    Optional<String> failureReason,
    Optional<Integer> agreementScore,
    Optional<String> agreementRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QueryStatus { RECEIVED, RUNNING, SYNTHESIZED, DEGRADED, BLOCKED, REJECTED }
```

### Events (on `QueryEntity`)

`QueryCreated`, `QueryRejected`, `QueryStarted`, `KeywordResultAttached`, `SemanticResultAttached`, `ChainOfThoughtResultAttached`, `AnswerSynthesized`, `QueryDegraded`, `QueryBlocked`, `AgreementScored`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/queries` — body `{ question, submittedBy? }` → `{ queryId }`. Starts a workflow.
- `GET /api/queries` — list all queries. Optional `?status=RECEIVED|RUNNING|SYNTHESIZED|DEGRADED|BLOCKED|REJECTED`.
- `GET /api/queries/{id}` — one query.
- `GET /api/queries/sse` — server-sent events stream of every query change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Strategy Workflow"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a query, live list of queries with status pills, expand-row to see the three strategy results, the synthesized answer, and the agreement score.

Browser title: `<title>Akka Sample: Multi-Strategy Workflow</title>`. Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid state-diagram label colours and edge-label `overflow:visible` per Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — input guardrail** (`before-agent-response` hook on the `QueryWorkflow` validate step): a deterministic checker rejects queries that are empty, exceed 2000 characters, or match a prohibited-content pattern. Blocking. Failure → `REJECTED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `EvalSampler` (TimedAction) picks one synthesized query every 5 minutes and asks `ConsistencyJudge` whether the three strategy answers agree with the synthesized answer, emitting an `AgreementScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `StrategyCoordinator` → `prompts/strategy-coordinator.md`. Decomposes the query into three strategy briefs; later synthesizes the three strategy results into one answer.
- `KeywordSearchAgent` → `prompts/keyword-search-agent.md`. Returns a `StrategyResult` on the keyword strategy.
- `SemanticRetrievalAgent` → `prompts/semantic-retrieval-agent.md`. Returns a `StrategyResult` on the semantic strategy.
- `ChainOfThoughtAgent` → `prompts/chain-of-thought-agent.md`. Returns a `StrategyResult` on the chain-of-thought strategy.
- `ConsistencyJudge` → `prompts/consistency-judge.md`. Returns an `AgreementVerdict` scoring cross-strategy agreement.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a query; processing progresses RECEIVED → RUNNING → SYNTHESIZED within 60 s; the expanded view shows three strategy results and one synthesized answer; UI reflects each transition via SSE.
2. **J2** — Submit a prohibited query; the query enters REJECTED before any strategy runs; no agent is called; the UI shows `failureReason`.
3. **J3** — Inject a strategy agent timeout (set one agent's step timeout to 1 s); query enters DEGRADED with the answer synthesized from the remaining strategies and `failureReason` naming the missing strategy.
4. **J4** — Inject a guardrail failure (Coordinator returns an answer with an empty summary); query enters BLOCKED with the strategy results still visible for audit.
5. **J5** — Wait one eval interval after a successful synthesis; the query row shows an agreement score (1–5) and rationale.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-strategy-workflow demonstrating the
debate-multi-perspective × general cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
debate-multi-perspective-general-multi-strategy-debate.
Java package io.akka.samples.multistrategyworkflow. Akka 3.6.0. HTTP port 9155.

Components to wire (exactly):
- 5 AutonomousAgents:
  * StrategyCoordinator — definition() with capability(TaskAcceptance.of(DECOMPOSE)
    .maxIterationsPerTask(2)) AND capability(TaskAcceptance.of(SYNTHESIZE)
    .maxIterationsPerTask(3)). System prompt loaded from prompts/strategy-coordinator.md.
    Returns StrategyBrief{keywordBrief, semanticBrief, chainOfThoughtBrief} for DECOMPOSE
    and SynthesizedAnswer{answer, summary, strategyResults, guardrailVerdict,
    synthesizedAt} for SYNTHESIZE.
  * KeywordSearchAgent — capability(TaskAcceptance.of(SEARCH_KEYWORD)
    .maxIterationsPerTask(2)). System prompt from prompts/keyword-search-agent.md.
    Returns StrategyResult{strategy="KEYWORD", answer, confidence, evidence:
    List<EvidenceItem{source, excerpt, relevanceScore}>, completedAt}.
  * SemanticRetrievalAgent — capability(TaskAcceptance.of(RETRIEVE_SEMANTIC)
    .maxIterationsPerTask(2)). System prompt from prompts/semantic-retrieval-agent.md.
    Returns StrategyResult{strategy="SEMANTIC", ...}.
  * ChainOfThoughtAgent — capability(TaskAcceptance.of(REASON_COT)
    .maxIterationsPerTask(2)). System prompt from prompts/chain-of-thought-agent.md.
    Returns StrategyResult{strategy="CHAIN_OF_THOUGHT", ...}.
  * ConsistencyJudge — capability(TaskAcceptance.of(SCORE_AGREEMENT)
    .maxIterationsPerTask(2)). System prompt from prompts/consistency-judge.md.
    Returns AgreementVerdict{score, rationale}.

- 1 deterministic helper QueryValidator (plain class in application/, NOT an
  agent): method validate(String question) -> ValidationResult{valid, reason}.
  Rejects empty questions, questions over 2000 characters, and questions matching
  a prohibited-content pattern (configurable keyword list in application.conf
  under multistrategy.guardrail.blocked-patterns). Pure, no LLM call.

- 1 Workflow QueryWorkflow with steps:
  createStep -> validateStep -> decomposeStep -> [parallel] keywordStep,
  semanticStep, chainOfThoughtStep -> joinStep -> synthesizeStep -> guardrailStep
  -> emitStep.
  createStep calls QueryEntity.createQuery emitting QueryCreated (RECEIVED).
  validateStep calls QueryValidator.validate(question); on failure calls
  QueryEntity.reject(reason) emitting QueryRejected (REJECTED) and ends workflow.
  decomposeStep calls forAutonomousAgent(StrategyCoordinator.class, DECOMPOSE)
  -> StrategyBrief. Then calls QueryEntity.markRunning emitting QueryStarted
  (status -> RUNNING).
  keywordStep, semanticStep, chainOfThoughtStep run in parallel (CompletionStage
  zip of three calls); each wrapped with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(60)). Each attaches its StrategyResult via
  KeywordResultAttached / SemanticResultAttached / ChainOfThoughtResultAttached.
  On any strategy agent timeout, transition to degradeStep that calls synthesizeStep
  with whichever results returned, then ends with QueryDegraded (DEGRADED) and
  failureReason naming the missing strategy.
  synthesizeStep calls forAutonomousAgent(StrategyCoordinator.class, SYNTHESIZE)
  with the available strategy results. guardrailStep runs the deterministic answer
  vetter (structural checks: non-empty answer, non-empty summary, >=1 strategy
  result, guardrailVerdict matches allowed set) and an LLM judge with a 5-second
  timeout; on failure, ends with QueryBlocked (BLOCKED). emitStep emits
  AnswerSynthesized (SYNTHESIZED).
  Override settings() with stepTimeout(60s) on the three strategy steps and the
  synthesizeStep, and defaultStepRecovery(maxRetries(2).failoverTo(error)).

- 1 EventSourcedEntity QueryEntity holding state Query{queryId, question,
  QueryStatus, Optional<StrategyResult> keywordResult,
  Optional<StrategyResult> semanticResult,
  Optional<StrategyResult> chainOfThoughtResult,
  Optional<SynthesizedAnswer> answer, Optional<String> failureReason,
  Optional<Integer> agreementScore, Optional<String> agreementRationale,
  Instant createdAt, Optional<Instant> finishedAt}.
  QueryStatus enum: RECEIVED, RUNNING, SYNTHESIZED, DEGRADED, BLOCKED, REJECTED.
  Events: QueryCreated, QueryRejected, QueryStarted, KeywordResultAttached,
  SemanticResultAttached, ChainOfThoughtResultAttached, AnswerSynthesized,
  QueryDegraded, QueryBlocked, AgreementScored.
  Commands: createQuery, reject, markRunning, attachKeyword, attachSemantic,
  attachChainOfThought, synthesize, degrade, block, recordAgreement, getQuery.
  emptyState() returns Query.initial("", "") with no commandContext() reference.

- 1 EventSourcedEntity QueryQueue with command enqueueQuery(question, submittedBy)
  emitting QueryReceived{queryId, question, submittedBy, receivedAt}.

- 1 View QueryView with row type QueryRow (mirrors Query minus the heavy
  evidence items; keep strategy answers/confidence and the overall answer but
  truncate evidence to counts for the list view). Table updater consumes
  QueryEntity events. ONE query getAllQueries SELECT * AS queries FROM
  query_view. No WHERE status filter — caller filters client-side.

- 1 Consumer QuerySimulatorConsumer subscribed to QueryQueue events; on
  QueryReceived starts a QueryWorkflow with the queryId as the workflow id,
  passing the question in the workflow start command.

- 2 TimedActions:
  * QuerySimulator — every 60s, reads next line from
    src/main/resources/sample-events/query-submissions.jsonl and calls
    QueryQueue.enqueueQuery.
  * EvalSampler — every 5 minutes, queries QueryView.getAllQueries, picks the
    oldest SYNTHESIZED query without an agreementScore, calls
    forAutonomousAgent(ConsistencyJudge.class, SCORE_AGREEMENT) with the three
    strategy results and the synthesized answer, then calls
    QueryEntity.recordAgreement(score, rationale).

- 2 HttpEndpoints:
  * QueryEndpoint at /api with POST /queries, GET /queries, GET /queries/{id},
    GET /queries/sse, and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/. Status filtering is client-side.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 Bootstrap (service-setup) that schedules the two TimedActions on startup
  and fails fast with a clear message if the configured model-provider key
  reference does not resolve (never echoing key material).

Companion files:
- QueryTasks.java declaring six Task<R> constants: DECOMPOSE (StrategyBrief),
  SEARCH_KEYWORD (StrategyResult), RETRIEVE_SEMANTIC (StrategyResult),
  REASON_COT (StrategyResult), SYNTHESIZE (SynthesizedAnswer), SCORE_AGREEMENT
  (AgreementVerdict).
- Domain records StrategyBrief, EvidenceItem, StrategyResult, SynthesizedAnswer,
  AgreementVerdict, QuerySubmission, ValidationResult.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9155 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also multistrategy.guardrail.blocked-patterns
  for the input validator.
- src/main/resources/sample-events/query-submissions.jsonl with 8 canned
  query lines covering a spread of topics (science, history, technology,
  reasoning puzzles) so the three strategies are visibly exercised.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 input guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  capability.*, model.*, and general domain metadata; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/strategy-coordinator.md, keyword-search-agent.md,
  semantic-retrieval-agent.md, chain-of-thought-agent.md, consistency-judge.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Strategy Workflow",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix
  on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Five tabs matching the formal
  exemplar: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7
  sub-tabs from governance.html with answers populated from risk-survey.yaml;
  unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live
  list with status pills; expand shows three strategy results, synthesized
  answer, agreement score). Browser title exactly:
  <title>Akka Sample: Multi-Strategy Workflow</title>. No subtitle on Overview.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone when
        the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface. Per-agent mock-response shapes for THIS blueprint:
    strategy-coordinator.json — list of either StrategyBrief or SynthesizedAnswer
      objects. 4–6 StrategyBrief entries (keywordBrief / semanticBrief /
      chainOfThoughtBrief triples for varied question types) and 4–6
      SynthesizedAnswer entries (each with a 60–120 word summary, three nested
      StrategyResult objects, guardrailVerdict = "ok").
    keyword-search-agent.json — 4–6 StrategyResult entries with strategy="KEYWORD",
      a confidence 0.0–1.0, and 2–4 EvidenceItem objects.
    semantic-retrieval-agent.json — 4–6 StrategyResult entries with strategy="SEMANTIC".
    chain-of-thought-agent.json — 4–6 StrategyResult entries with
      strategy="CHAIN_OF_THOUGHT", confidence, and a reasoning-step excerpt.
    consistency-judge.json — 4–6 AgreementVerdict entries (score 1–5 + one-sentence
      rationale on whether the strategies agree with the synthesized answer).
- A MockModelProvider.seedFor(queryId) helper makes the selection deterministic
  per query id so the same query in dev produces the same output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- (Lesson 1) AutonomousAgent never silently downgraded to Agent.
- (Lesson 4) Workflow step timeouts set explicitly on every agent-calling step
  (60s on the three strategy steps and the synthesize step).
- (Lesson 6) Optional<T> for every nullable lifecycle field on Query and QueryRow.
- (Lesson 7) AutonomousAgent requires the companion QueryTasks.java.
- (Lesson 8) Verify model names against the provider's current lineup before
  locking them in application.conf.
- (Lesson 9) Run command is "/akka:build", never "mvn akka:run".
- (Lesson 10) Port 9155 declared explicitly in application.conf.
- (Lesson 11) Source-platform metadata is corpus-internal — never user-facing.
- (Lesson 12) UI fits the 1080px content column with no horizontal scroll.
- (Lesson 13) Integration tier labels are descriptive ("Runs out of the box"),
  never T1/T2/T3/T4 and never the word "deferred".
- (Lesson 23) No competitor brand names anywhere in user-facing text.
- (Lesson 24) static-resources/index.html includes the mermaid CSS overrides
  AND theme variables (state-diagram label colour white, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- (Lesson 25) Never write the model-provider key value to disk.
- (Lesson 26) Tab switching by data-tab / data-panel attribute; exactly five
  .tab-panel sections; no zombie panels.
- Parallel strategy steps use CompletionStage zip, NOT sequential calls.
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var
  export block.
- No forbidden words in user-facing text.
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
