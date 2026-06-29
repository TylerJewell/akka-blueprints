# SPEC — ragas-eval-harness

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** RAGAS Evaluation for Agents.
**One-line pitch:** Submit a question; a retrieval agent answers it from a document corpus; a RAGAS evaluator scores faithfulness, answer relevance, and context precision; the two iterate until the answer passes all thresholds or the harness exhausts its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern applied to RAG quality assurance: a Workflow alternates between a generator agent (`RetrievalAgent`) and a reviewer agent (`RagasEvaluatorAgent`), feeding each score set back into the next answer attempt until every RAGAS metric clears its threshold or a halt occurs. The blueprint wires two governance mechanisms — a **continuous-accuracy eval** that records every cycle's metric scores for trend analysis, and a **CI gate** that blocks a deployment when the rolling pass rate over the most recent evaluation window falls below a configured floor.

## 3. User-facing flows

The user opens the App UI tab and submits a question (free text, optional corpus-filter tag).

1. The system creates an `EvalRun` record in `ANSWERING` and starts an `EvalWorkflow`.
2. The Retrieval agent fetches the top-k context chunks from the in-memory corpus that match the question, composes a grounded answer, and returns an `AnswerRecord`.
3. The score-floor guardrail checks the answer's self-reported grounding confidence against a deterministic minimum. Under-threshold answers are short-circuited back to the Retrieval agent with a structured feedback note; they never reach the RAGAS Evaluator.
4. The RAGAS Evaluator scores the (question, answer, retrieved-context) triple against three metrics: faithfulness (does every claim in the answer appear in the context?), answer relevance (does the answer address the question?), and context precision (are the retrieved chunks relevant to the question?). It returns either `PASS` with per-metric scores, or `RETRY` with a `RagasFeedback` payload naming which metrics failed and why.
5. On `PASS`, the workflow transitions the run to `PASSED` with the winning attempt's answer and the evaluator's scores.
6. On `RETRY`, the workflow records the attempt, the guardrail verdict, the scores, and the feedback on the entity, then calls the Retrieval agent again with the feedback attached. The Retrieval agent produces attempt #2, typically adjusting its retrieval query or answer composition strategy.
7. If the loop reaches `maxAttempts` (default 3) without a `PASS`, the halt mechanism activates: the workflow ends with `FAILED_FINAL`, the highest-scoring attempt is preserved on the entity along with every score set for audit, and a `ScoreRecorded` event captures the terminal outcome.

A `QuestionSimulator` (TimedAction) drips a canned question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RetrievalAgent` | `AutonomousAgent` | Answers a question by retrieving context chunks and composing a grounded response; accepts prior RAGAS feedback on revisions. | `EvalWorkflow` | returns `AnswerRecord` to workflow |
| `RagasEvaluatorAgent` | `AutonomousAgent` | Scores a (question, answer, context) triple against three RAGAS metrics; returns `PASS` or `RETRY` with feedback. | `EvalWorkflow` | returns `RagasScore` to workflow |
| `EvalWorkflow` | `Workflow` | Runs the answer → guardrail → evaluate → revise loop; halts at the ceiling. | `EvalEndpoint`, `QuestionConsumer` | `EvalRunEntity` |
| `EvalRunEntity` | `EventSourcedEntity` | Holds the evaluation run lifecycle, every attempt, every score set, and the final outcome. | `EvalWorkflow` | `EvalRunsView` |
| `QuestionQueue` | `EventSourcedEntity` | Logs each submitted question for replay and audit. | `EvalEndpoint`, `QuestionSimulator` | `QuestionConsumer` |
| `EvalRunsView` | `View` | List-of-runs read model. | `EvalRunEntity` events | `EvalEndpoint` |
| `QuestionConsumer` | `Consumer` | Subscribes to `QuestionQueue` events; starts a workflow per submission. | `QuestionQueue` events | `EvalWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample question every 60 s from `sample-events/eval-questions.jsonl`. | scheduler | `QuestionQueue` |
| `ScoreSampler` | `TimedAction` | Every 30 s, scans `EvalRunsView`, records a `ScoreRecorded` event for any cycle that completed since the last tick. | scheduler | `EvalRunEntity` |
| `EvalEndpoint` | `HttpEndpoint` | `/api/runs/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `EvalRunsView`, `QuestionQueue`, `EvalRunEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Question(String text, Optional<String> corpusTag, String submittedBy) {}

record ContextChunk(String chunkId, String sourceDoc, String text, double relevanceScore) {}

record AnswerRecord(
    String text,
    List<ContextChunk> retrievedChunks,
    double groundingConfidence,
    Instant answeredAt
) {}

record GroundingVerdict(boolean passed, String reasonCode, String detail) {}

record RagasFeedback(
    List<String> failedMetrics,
    Map<String, Double> metricScores,
    String overallRationale
) {}

record RagasScore(
    EvalVerdict verdict,
    RagasFeedback feedback,
    double faithfulness,
    double answerRelevance,
    double contextPrecision,
    Instant evaluatedAt
) {}

record EvalAttempt(
    int attemptNumber,
    AnswerRecord answer,
    GroundingVerdict grounding,
    Optional<RagasScore> score
) {}

record EvalRun(
    String runId,
    String questionText,
    Optional<String> corpusTag,
    int maxAttempts,
    EvalRunStatus status,
    List<EvalAttempt> attempts,
    Optional<Integer> passedAttemptNumber,
    Optional<String> passedAnswerText,
    Optional<String> failureReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EvalRunStatus { ANSWERING, EVALUATING, PASSED, FAILED_FINAL }

enum EvalVerdict { PASS, RETRY }
```

### Events (on `EvalRunEntity`)

`RunCreated`, `AttemptAnswered`, `AttemptGroundingVerdictRecorded`, `AttemptScored`, `RunPassed`, `RunFailedFinal`, `ScoreRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/runs` — body `{ question, corpusTag?, submittedBy? }` → `{ runId }`. Starts a workflow.
- `GET /api/runs` — list all runs. Optional `?status=ANSWERING|EVALUATING|PASSED|FAILED_FINAL`.
- `GET /api/runs/{id}` — one run (including every attempt and every score set).
- `GET /api/runs/sse` — server-sent events stream of every run change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "RAGAS Evaluation for Agents"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-periodic = blue, ci-gate = orange).
- **App UI** — form to submit a question, live list of evaluation runs with status pills, click-to-expand per-attempt timeline showing each answer, the grounding verdict, the evaluator's verdict, per-metric scores, and feedback.

Browser title: `<title>Akka Sample: RAGAS Evaluation for Agents</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **E1 — eval-periodic** (`on-decision-eval`): every cycle's RAGAS score set is recorded as a `ScoreRecorded` event with `{ attemptNumber, verdict, faithfulness, answerRelevance, contextPrecision, ceilingExceeded }`. The `ScoreSampler` TimedAction is the canonical writer; the workflow itself also emits one on terminal transitions. Enforcement: non-blocking. Events surface in the App UI's per-attempt timeline and in `/api/runs/{id}`.
- **CI1 — ci-gate** (`test-gate`): before the service artifact is promoted, a gate queries `EvalRunsView` for the rolling pass rate over the most recent 100 evaluation runs. If the rate is below `ragas-eval.ci.min-pass-rate` (default 0.80), the gate emits a `CiGateFailed` event and returns a non-zero exit code, blocking the promotion pipeline. Enforcement: build-gate.

## 9. Agent prompts

- `RetrievalAgent` → `prompts/retrieval-agent.md`. Retrieves top-k context chunks from the in-memory corpus; composes a grounded answer; on a revision call, takes the prior `RagasFeedback` as input and refines retrieval strategy or answer composition.
- `RagasEvaluatorAgent` → `prompts/ragas-evaluator-agent.md`. Scores the (question, answer, retrieved-context) triple against the three RAGAS metrics; returns `PASS` with per-metric scores or `RETRY` with a `RagasFeedback` payload.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a question; run progresses `ANSWERING` → `EVALUATING` → `PASSED` within the retry ceiling; the App UI shows every attempt's answer and score set.
2. **J2 — halt at ceiling** — Submit a question whose context is absent from the corpus (test mode forces the Evaluator to `RETRY` every attempt); run progresses through every attempt and lands in `FAILED_FINAL` with the best-scoring attempt preserved and a structured failure reason.
3. **J3 — grounding guardrail** — Submit a question where the retrieval confidence is below the minimum; the guardrail short-circuits with `reasonCode = BELOW_CONFIDENCE_FLOOR`, the Retrieval agent re-answers with a refined query, the cycle continues.
4. **J4 — score-event timeline** — The expanded view of any run shows one `ScoreRecorded` event per scored attempt and one terminal event on completion.
5. **J5 — CI gate** — After injecting 25 failed runs into the view (pass rate < 0.80), calling `GET /api/ci/gate-status` returns `{ "gatePassed": false, "passRate": ... }`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ragas-eval-harness demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-ragas-eval-harness.
Java package io.akka.samples.ragasevaluationforagents. Akka 3.6.0.
HTTP port 9873.

Components to wire (exactly):
- 2 AutonomousAgents:
  * RetrievalAgent — definition() with
    capability(TaskAcceptance.of(ANSWER).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ANSWER).maxIterationsPerTask(3)).
    System prompt loaded from prompts/retrieval-agent.md. Returns
    AnswerRecord{text, retrievedChunks, groundingConfidence, answeredAt}
    for both ANSWER and REVISE_ANSWER. The REVISE_ANSWER task takes
    (originalQuestion, priorAnswer, RagasFeedback) as inputs.
  * RagasEvaluatorAgent — definition() with
    capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)). System
    prompt from prompts/ragas-evaluator-agent.md. Returns
    RagasScore{verdict, feedback, faithfulness, answerRelevance,
    contextPrecision, evaluatedAt} where verdict is the EvalVerdict enum
    (PASS | RETRY).

- 1 Workflow EvalWorkflow with steps:
    startStep -> answerStep -> groundingStep -> [grounding FAIL? answerStep
    again with structured feedback : scoreStep] ->
    [verdict PASS? passStep : (attemptCount < maxAttempts ?
       answerStep with feedback attached : failStep)] -> END.
  answerStep calls forAutonomousAgent(RetrievalAgent.class, runId)
    .runSingleTask(ANSWER or REVISE_ANSWER) then forTask(taskId)
    .result(ANSWER or REVISE_ANSWER). scoreStep calls
    forAutonomousAgent(RagasEvaluatorAgent.class, runId)
    .runSingleTask(EVALUATE). passStep emits RunPassed. failStep emits
    RunFailedFinal with the highest-scoring attempt's text as best-of and
    a structured failureReason. Override settings() with stepTimeout(60s)
    on answerStep and scoreStep, and
    defaultStepRecovery(maxRetries(2).failoverTo(failStep)).
  groundingStep is a pure-function step (no LLM call): checks
    answer.groundingConfidence() >= ragas-eval.grounding.min-confidence
    (default 0.4). On FAIL, emits AttemptGroundingVerdictRecorded with
    verdict.passed = false and reasonCode = "BELOW_CONFIDENCE_FLOOR",
    then transitions back to answerStep with a structured feedback
    RagasFeedback containing failedMetrics=["grounding"] and
    overallRationale("Answer grounding confidence below minimum floor;
    refine retrieval query and resubmit.").

- 1 EventSourcedEntity EvalRunEntity holding state EvalRun{runId,
  questionText, Optional<String> corpusTag, maxAttempts,
  EvalRunStatus status, List<EvalAttempt> attempts,
  Optional<Integer> passedAttemptNumber, Optional<String> passedAnswerText,
  Optional<String> failureReason, Instant createdAt,
  Optional<Instant> finishedAt}. EvalRunStatus enum: ANSWERING, EVALUATING,
  PASSED, FAILED_FINAL. Events: RunCreated, AttemptAnswered,
  AttemptGroundingVerdictRecorded, AttemptScored, RunPassed, RunFailedFinal,
  ScoreRecorded. Commands: createRun, recordAnswer, recordGrounding,
  recordScore, pass, failFinal, recordEval, getRun. emptyState() returns
  EvalRun.initial("", "", 3) with no commandContext() reference.
  Event-applier wraps lifecycle fields with Optional.of(...).

- 1 EventSourcedEntity QuestionQueue with command submitQuestion(text,
  corpusTag, submittedBy) emitting QuestionSubmitted{runId, text,
  corpusTag, submittedBy, submittedAt}.

- 1 View EvalRunsView with row type EvalRunRow (mirrors EvalRun; the
  attempts list is preserved as-is — the list is bounded at maxAttempts
  so size stays reasonable). Table updater consumes EvalRunEntity events.
  ONE query getAllRuns SELECT * AS runs FROM eval_runs_view. No WHERE
  status filter — caller filters client-side because Akka cannot
  auto-index enum columns (Lesson 2).

- 1 Consumer QuestionConsumer subscribed to QuestionQueue events; on
  QuestionSubmitted starts an EvalWorkflow with the runId as the
  workflow id.

- 2 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/eval-questions.jsonl and calls
    QuestionQueue.submitQuestion.
  * ScoreSampler — every 30s, queries EvalRunsView.getAllRuns, finds runs
    with a scored attempt that has not yet been recorded as a
    ScoreRecorded event, and calls EvalRunEntity.recordEval(attemptNumber,
    verdict, faithfulness, answerRelevance, contextPrecision,
    ceilingExceeded). Idempotent per (runId, attemptNumber).

- 2 HttpEndpoints:
  * EvalEndpoint at /api with POST /runs, GET /runs, GET /runs/{id},
    GET /runs/sse, GET /ci/gate-status, and three /api/metadata/*
    endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /runs body is
    {question, corpusTag?, submittedBy?}; missing corpusTag defaults to
    null, missing submittedBy defaults to "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- EvalTasks.java declaring three Task<R> constants: ANSWER (resultConformsTo
  AnswerRecord), REVISE_ANSWER (AnswerRecord), EVALUATE (RagasScore).
- Domain records AnswerRecord, ContextChunk, GroundingVerdict,
  RagasFeedback, RagasScore, EvalAttempt, EvalRun; enums EvalRunStatus,
  EvalVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9873 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  the canonical env vars (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  ragas-eval.workflow.max-attempts = 3,
  ragas-eval.grounding.min-confidence = 0.4, and
  ragas-eval.ci.min-pass-rate = 0.80, all overridable by env var.
- src/main/resources/sample-events/eval-questions.jsonl with 8 canned
  question lines, each shaped {"text":"...", "corpusTag":null}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from
  classpath).
- eval-matrix.yaml at the project root with 2 controls (E1 eval-periodic
  on-decision-eval, CI1 ci-gate test-gate) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = rag-quality-evaluation,
  decisions.authority_level = advisory-scores, data.data_classes.pii = false,
  capabilities.content-generation = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/retrieval-agent.md, prompts/ragas-evaluator-agent.md loaded at
  agent startup as system prompts.
- README.md at the project root: title "Akka Sample: RAGAS Evaluation for
  Agents", one-line pitch, prerequisites, generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
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
  <title>Akka Sample: RAGAS Evaluation for Agents</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets
        model-provider = mock.
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
  src/main/resources/mock-responses/<agent-name>.json (one file per agent:
  retrieval-agent.json, ragas-evaluator-agent.json), picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed
  return shape.
- Per-agent mock-response shapes for THIS blueprint:
    retrieval-agent.json — 6 AnswerRecord entries. Three have
      groundingConfidence >= 0.7 and 2-3 relevant ContextChunk items.
      Two are "revision" answers that reference the prior RagasFeedback
      (different retrieval strategy, tighter answer). One has
      groundingConfidence = 0.2 to exercise the grounding guardrail in J3.
    ragas-evaluator-agent.json — 6 RagasScore entries. Three return
      verdict=PASS with faithfulness >= 0.85, answerRelevance >= 0.80,
      contextPrecision >= 0.75. Three return verdict=RETRY with at least
      one metric below threshold and a RagasFeedback naming the failing
      metrics.
- A MockModelProvider.seedFor(runId, attemptNumber) helper makes the
  selection deterministic per (runId, attemptNumber).

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
  RetrievalAgent and RagasEvaluatorAgent both extend
  akka.javasdk.agent.autonomous.AutonomousAgent and ship with an
  EvalTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never
  inherited.
- Lesson 6: every nullable lifecycle field on the EvalRun row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: EvalTasks.java is mandatory; generating RetrievalAgent or
  RagasEvaluatorAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9873, declared in application.conf
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
