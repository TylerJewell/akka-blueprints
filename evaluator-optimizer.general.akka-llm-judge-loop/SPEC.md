# SPEC — llm-judge-loop

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** LLM-as-a-Judge Evaluator-Optimizer.
**One-line pitch:** Submit a question; a generator agent produces an answer; a judge agent scores it against a rubric; the two iterate until the judge accepts or the loop hits its retry ceiling.

## 2. What this blueprint demonstrates

The **evaluator-optimizer** coordination pattern wired with Akka's first-party primitives: a Workflow alternates between a generator agent (`GeneratorAgent`) and a reviewer agent (`JudgeAgent`), feeding each judgment back into the next answer until convergence or a halt. The blueprint also demonstrates two governance mechanisms — an **eval-event** that records every cycle's verdict for downstream quality measurement, and an **output guardrail** that gates each generated answer against a structural check before the judge runs.

## 3. User-facing flows

The user opens the App UI tab and submits a question (free text, plus an optional domain tag and score threshold).

1. The system creates an `Evaluation` record in `GENERATING` and starts an `EvaluationWorkflow`.
2. The Generator produces answer #1: a well-reasoned response to the question.
3. The output guardrail vets the answer against a structural check (minimum token count, no refused-answer sentinel). Answers failing the check are short-circuited back to the Generator with a deterministic feedback note; they never reach the Judge.
4. The Judge scores the answer against a fixed rubric (accuracy, completeness, coherence, groundedness) and returns either `ACCEPT` with a one-line rationale, or `REVISE` with a typed `JudgeFeedback` payload (up to three bullets).
5. On `ACCEPT`, the workflow transitions the evaluation to `ACCEPTED` with the winning answer and the judge's rationale.
6. On `REVISE`, the workflow records the attempt, the guardrail verdict, the judgment, and the judge's verdict on the entity, then calls the Generator again with the feedback attached. The Generator produces answer #2.
7. If the loop reaches `maxAttempts` (default 4) without an `ACCEPT`, the halt mechanism activates: the workflow ends with `REJECTED_FINAL`, the highest-scoring answer is preserved on the entity along with every judgment for audit, and a `JudgmentRecorded` event captures the rejection.

A `QuestionSimulator` (TimedAction) drips a canned question every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `GeneratorAgent` | `AutonomousAgent` | Produces an answer to a question; accepts prior judge feedback on revisions. | `EvaluationWorkflow` | returns `GeneratedAnswer` to workflow |
| `JudgeAgent` | `AutonomousAgent` | Scores an answer against the rubric; returns `ACCEPT` or `REVISE` with feedback. | `EvaluationWorkflow` | returns `Judgment` to workflow |
| `EvaluationWorkflow` | `Workflow` | Runs the generate → guardrail → judge → revise loop; halts at the ceiling. | `EvaluationEndpoint`, `QuestionConsumer` | `EvaluationEntity` |
| `EvaluationEntity` | `EventSourcedEntity` | Holds the evaluation lifecycle, every attempt, every judgment, and the final outcome. | `EvaluationWorkflow` | `EvaluationsView` |
| `QuestionQueue` | `EventSourcedEntity` | Logs each submitted question for replay and audit. | `EvaluationEndpoint`, `QuestionSimulator` | `QuestionConsumer` |
| `EvaluationsView` | `View` | List-of-evaluations read model. | `EvaluationEntity` events | `EvaluationEndpoint` |
| `QuestionConsumer` | `Consumer` | Subscribes to `QuestionQueue` events; starts a workflow per submission. | `QuestionQueue` events | `EvaluationWorkflow` |
| `QuestionSimulator` | `TimedAction` | Drips a sample question every 60 s from `sample-events/judge-questions.jsonl`. | scheduler | `QuestionQueue` |
| `JudgeSampler` | `TimedAction` | Every 30 s, scans `EvaluationsView`, records a `JudgmentRecorded` event for any cycle that completed since the last tick. | scheduler | `EvaluationEntity` |
| `EvaluationEndpoint` | `HttpEndpoint` | `/api/evaluations/*` — submit, get, list, SSE; plus `/api/metadata/*`. | — | `EvaluationsView`, `QuestionQueue`, `EvaluationEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI; redirects `/` to `/app/index.html`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6).

### Records

```java
record Question(String questionText, String domainTag, int scoreThreshold, String submittedBy) {}

record GeneratedAnswer(String text, int tokenCount, Instant generatedAt) {}

record GuardrailVerdict(boolean passed, String reasonCode, String detail) {}

record JudgeFeedback(List<String> bullets, String overallRationale) {}

record Judgment(JudgeVerdict verdict, JudgeFeedback feedback, int score, Instant evaluatedAt) {}

record Attempt(
    int attemptNumber,
    GeneratedAnswer answer,
    GuardrailVerdict guardrail,
    Optional<Judgment> judgment
) {}

record Evaluation(
    String evaluationId,
    String questionText,
    String domainTag,
    int scoreThreshold,
    int maxAttempts,
    EvaluationStatus status,
    List<Attempt> attempts,
    Optional<Integer> acceptedAttemptNumber,
    Optional<String> acceptedText,
    Optional<String> rejectionReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum EvaluationStatus { GENERATING, JUDGING, ACCEPTED, REJECTED_FINAL }

enum JudgeVerdict { ACCEPT, REVISE }
```

### Events (on `EvaluationEntity`)

`EvaluationCreated`, `AttemptGenerated`, `AttemptGuardrailVerdictRecorded`, `AttemptJudged`, `EvaluationAccepted`, `EvaluationRejectedFinal`, `JudgmentRecorded`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/evaluations` — body `{ questionText, domainTag?, scoreThreshold?, submittedBy? }` → `{ evaluationId }`. Starts a workflow.
- `GET /api/evaluations` — list all evaluations. Optional `?status=GENERATING|JUDGING|ACCEPTED|REJECTED_FINAL`.
- `GET /api/evaluations/{id}` — one evaluation (including every attempt and every judgment).
- `GET /api/evaluations/sse` — server-sent events stream of every evaluation change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — single self-contained `static-resources/index.html`.

## 7. UI

The five-tab structure inherited from the formal exemplar — see `reference/ui-mockup.md`.

- **Overview** — eyebrow "Overview" + headline "LLM-as-a-Judge Evaluator-Optimizer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) with Akka theme variables and the Lesson 24 CSS overrides for state-diagram labels and edge-label `foreignObject` overflow.
- **Risk Survey** — 7 sub-tabs in the `governance.html` style with answers from `risk-survey.yaml`; deployer-specific fields faded.
- **Eval Matrix** — 5-column table with click-to-expand rows; mechanism-coloured ID badges (eval-event = blue, guardrail = red).
- **App UI** — form to submit a question, live list of evaluations with status pills, click-to-expand per-attempt timeline showing each answer, the guardrail verdict, the judge's verdict, and the judge's feedback.

Browser title: `<title>Akka Sample: LLM-as-a-Judge Evaluator-Optimizer</title>`.

Tab switching MUST be attribute-based (`data-tab` / `data-panel`), never NodeList-index-based — see Lesson 26. The DOM contains exactly five `<section class="tab-panel">` elements; any removed panel must be deleted from the HTML, not hidden with `display:none`.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`after-llm-response` on `GeneratorAgent`): a deterministic structural check applied to each generated answer — minimum token count check and absence of a refused-answer sentinel string. Answers failing the check short-circuit back to the Generator with a structured feedback note (`reasonCode = STRUCTURAL_FAIL`); they never reach the Judge. Enforcement: blocking.
- **E1 — eval-event** (`on-decision-eval`): every cycle's judgment is recorded as a `JudgmentRecorded` event with `{ attemptNumber, verdict, score, guardrailFailed }`. The `JudgeSampler` TimedAction is the canonical writer; the workflow itself also emits an event on terminal transitions. Enforcement: non-blocking. The events surface in the App UI's per-attempt timeline and in `/api/evaluations/{id}`.

## 9. Agent prompts

- `GeneratorAgent` → `prompts/generator.md`. Produces a reasoned answer to the question; on a revision call, takes the prior `JudgeFeedback` as input and produces an improved answer.
- `JudgeAgent` → `prompts/judge.md`. Scores an answer against the fixed rubric; returns `ACCEPT` with a one-line rationale or `REVISE` with up to three specific bullets.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1 — convergence** — Submit a question; evaluation progresses `GENERATING` → `JUDGING` → `ACCEPTED` within the retry ceiling; the App UI shows every attempt's answer and judgment.
2. **J2 — halt at ceiling** — Submit a question whose rubric is impossible to satisfy (test mode forces the Judge to `REVISE` every attempt); evaluation progresses through every attempt and lands in `REJECTED_FINAL` with the best answer preserved.
3. **J3 — guardrail block** — Submit a question that triggers the structural check failure; the Generator's first answer fails the guardrail; the Generator re-answers; the cycle continues.
4. **J4 — eval-event timeline** — The expanded view of any evaluation shows one `JudgmentRecorded` event per attempt and one terminal event on completion.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named llm-judge-loop demonstrating the evaluator-optimizer ×
general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Artifact id evaluator-optimizer-general-akka-llm-judge-loop.
Java package io.akka.samples.llmasajudgeevaluatoroptimizer.
Akka 3.6.0. HTTP port 9358.

Components to wire (exactly):
- 2 AutonomousAgents:
  * GeneratorAgent — definition() with
    capability(TaskAcceptance.of(GENERATE).maxIterationsPerTask(3))
    AND capability(TaskAcceptance.of(REVISE_ANSWER).maxIterationsPerTask(3)).
    System prompt loaded from prompts/generator.md. Returns GeneratedAnswer{text,
    tokenCount, generatedAt} for both GENERATE and REVISE_ANSWER. The
    REVISE_ANSWER task takes (originalQuestion, priorAnswer, JudgeFeedback) as
    inputs.
  * JudgeAgent — definition() with
    capability(TaskAcceptance.of(JUDGE).maxIterationsPerTask(2)). System
    prompt from prompts/judge.md. Returns Judgment{verdict, feedback, score,
    evaluatedAt} where verdict is the JudgeVerdict enum (ACCEPT | REVISE)
    and score is a 1–5 integer rubric.

- 1 Workflow EvaluationWorkflow with steps:
    startStep -> generateStep -> guardrailStep -> [guardrail FAIL?
    generateStep again with structured feedback : judgeStep] ->
    [verdict ACCEPT? acceptStep : (attemptCount < maxAttempts ?
       generateStep with feedback attached : rejectStep)] -> END.
  generateStep calls forAutonomousAgent(GeneratorAgent.class, evaluationId)
    .runSingleTask(GENERATE or REVISE_ANSWER) then
    forTask(taskId).result(GENERATE or REVISE_ANSWER). judgeStep calls
    forAutonomousAgent(JudgeAgent.class, evaluationId).runSingleTask(JUDGE).
    acceptStep emits EvaluationAccepted.
    rejectStep emits EvaluationRejectedFinal with the highest-scoring
    attempt's text as best-of and a structured rejectionReason.
    Override settings() with stepTimeout(60s) on generateStep and
    judgeStep, and defaultStepRecovery(maxRetries(2).failoverTo(rejectStep)).
  guardrailStep is a pure-function step (no LLM call): checks
    answer.tokenCount() >= minTokens AND !answer.text().contains(REFUSED_SENTINEL).
    On FAIL, emits AttemptGuardrailVerdictRecorded with
    verdict.passed = false and reasonCode = "STRUCTURAL_FAIL", then
    transitions back to generateStep with a structured feedback
    JudgeFeedback("Answer did not meet structural requirements; provide a
    complete response.").

- 1 EventSourcedEntity EvaluationEntity holding state
  Evaluation{evaluationId, questionText, domainTag, scoreThreshold,
  maxAttempts, EvaluationStatus status, List<Attempt> attempts,
  Optional<Integer> acceptedAttemptNumber, Optional<String> acceptedText,
  Optional<String> rejectionReason, Instant createdAt,
  Optional<Instant> finishedAt}.
  EvaluationStatus enum: GENERATING, JUDGING, ACCEPTED, REJECTED_FINAL.
  Events: EvaluationCreated, AttemptGenerated,
  AttemptGuardrailVerdictRecorded, AttemptJudged, EvaluationAccepted,
  EvaluationRejectedFinal, JudgmentRecorded.
  Commands: createEvaluation, recordAnswer, recordGuardrail, recordJudgment,
  accept, rejectFinal, recordJudgmentEvent, getEvaluation.
  emptyState() returns Evaluation.initial("", "", "", 4, 4) with no
  commandContext() reference. Event-applier wraps lifecycle fields with
  Optional.of(...).

- 1 EventSourcedEntity QuestionQueue with command submitQuestion(questionText,
  domainTag, scoreThreshold, submittedBy) emitting
  QuestionSubmitted{evaluationId, questionText, domainTag, scoreThreshold,
  submittedBy, submittedAt}.

- 1 View EvaluationsView with row type EvaluationRow (mirrors Evaluation;
  the attempts list is preserved as-is — bounded at maxAttempts so size stays
  reasonable). Table updater consumes EvaluationEntity events. ONE query
  getAllEvaluations SELECT * AS evaluations FROM evaluations_view. No WHERE
  status filter — caller filters client-side because Akka cannot auto-index
  enum columns (Lesson 2).

- 1 Consumer QuestionConsumer subscribed to QuestionQueue events; on
  QuestionSubmitted starts an EvaluationWorkflow with the evaluationId as
  the workflow id.

- 2 TimedActions:
  * QuestionSimulator — every 60s, reads next line from
    src/main/resources/sample-events/judge-questions.jsonl and calls
    QuestionQueue.submitQuestion.
  * JudgeSampler — every 30s, queries EvaluationsView.getAllEvaluations,
    finds evaluations with a judged attempt that has not yet been recorded as
    a JudgmentRecorded event, and calls
    EvaluationEntity.recordJudgmentEvent(attemptNumber, verdict, score,
    guardrailFailed). Idempotent per (evaluationId, attemptNumber).

- 2 HttpEndpoints:
  * EvaluationEndpoint at /api with POST /evaluations,
    GET /evaluations, GET /evaluations/{id}, GET /evaluations/sse, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. The POST /evaluations body is
    {questionText, domainTag?, scoreThreshold?, submittedBy?}; missing
    scoreThreshold defaults to 4, missing submittedBy defaults to
    "anonymous".
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- JudgeTasks.java declaring three Task<R> constants: GENERATE (resultConformsTo
  GeneratedAnswer), REVISE_ANSWER (GeneratedAnswer), JUDGE (Judgment).
- Domain records GeneratedAnswer, GuardrailVerdict, JudgeFeedback, Judgment,
  Attempt, Evaluation; enums EvaluationStatus, JudgeVerdict.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9358 and
  akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from the canonical env vars
  (${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}). Per-workflow config
  llm-judge-loop.evaluation.max-attempts = 4 and
  llm-judge-loop.evaluation.min-token-count = 50, overridable by env var.
- src/main/resources/sample-events/judge-questions.jsonl with 8 canned
  question lines, each shaped
  {"questionText":"...", "domainTag":"general", "scoreThreshold":4}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  after-llm-response, E1 eval-event on-decision-eval) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling
  purpose.primary_function = answer-quality-evaluation,
  decisions.authority_level = draft-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/generator.md, prompts/judge.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: LLM-as-a-Judge
  Evaluator-Optimizer", one-line pitch, prerequisites, generate-the-system,
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
  <title>Akka Sample: LLM-as-a-Judge Evaluator-Optimizer</title>.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If
  exactly one is set, default application.conf's model-provider to match
  and proceed silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
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
  generator.json, judge.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    generator.json — 6 GeneratedAnswer entries. Three are "first-pass"
      answers with tokenCount between 80 and 200. Two are "revision" answers
      with higher token counts that directly reference the prior feedback.
      One is an intentionally malformed answer (tokenCount < 50) used to
      exercise the guardrail in J3.
    judge.json — 6 Judgment entries. Three return verdict=ACCEPT with
      score=4 or 5 and a one-sentence rationale. Three return
      verdict=REVISE with score=2 or 3 and a JudgeFeedback payload of
      three bullets ("answer lacks supporting evidence", "coherence breaks
      in second paragraph", "groundedness score insufficient").
- A MockModelProvider.seedFor(evaluationId, attemptNumber) helper makes the
  selection deterministic per (evaluationId, attemptNumber) so the same
  evaluation in dev produces the same loop trajectory across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent. GeneratorAgent
  and JudgeAgent both extend akka.javasdk.agent.autonomous.AutonomousAgent
  and ship with a JudgeTasks companion declaring the three Task<R> constants.
- Lesson 4: every workflow step that calls an agent has an explicit
  stepTimeout(60s) override; the default 5-second timeout is never inherited.
- Lesson 6: every nullable lifecycle field on the Evaluation row record is
  Optional<T>; the event-applier wraps values with Optional.of(...).
- Lesson 7: JudgeTasks.java is mandatory; generating GeneratorAgent or
  JudgeAgent without it is a compile error.
- Lesson 8: model names are claude-sonnet-4-6, gpt-4o, gemini-2.5-flash —
  verified current; do not substitute deprecated identifiers.
- Lesson 9: the run command is /akka:build (Claude Code slash command),
  never mvn akka:run.
- Lesson 10: HTTP port is 9358, declared in application.conf
  dev-mode.http-port.
- Lesson 11: source.platform is corpus-internal; the generated UI never
  surfaces a competitor brand name.
- Lesson 12: the App UI fits the 1080px content column with no horizontal
  scroll.
- Lesson 13: integration tier is shown as "Runs out of the box" — never
  T1/T2/T3/T4, never the word "deferred".
- Lesson 23: forbidden words do not appear in any user-facing surface.
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
