# SPEC — react-math-wiki

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MathWikiReAct.
**One-line pitch:** A user submits a factual or numeric question; one AI agent runs a ReAct loop — deciding step-by-step whether to call `evaluate_math` or `search_wikipedia` — and returns a typed `ResearchAnswer` with the answer text, the tool-call trace, and a confidence tag.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the research-intel domain. One `ResearchAgent` (AutonomousAgent) drives the entire ReAct loop; surrounding components only prepare its input, record its trace, and audit its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** validates every `evaluate_math` argument against a restricted expression grammar before the tool executes — preventing arbitrary code (file I/O, network calls, reflection) from running under the guise of a math computation. The agent may call `search_wikipedia` freely; only code-evaluation arguments pass through the guardrail check.

The blueprint shows that even a tool-calling agent with two tools requires targeted sandboxing when one of those tools executes code. The guardrail fires per call, not per response, giving the agent full freedom on the safe tool while enforcing boundaries on the risky one.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Question** textarea (or picks one of five seeded examples — three word-problems, two factual lookups).
2. The user clicks **Ask**. The UI POSTs to `/api/questions` and receives a `questionId`.
3. The question card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `RUNNING` — the agent's ReAct loop has started.
4. Each tool call the agent makes emits a `ToolCallRecorded` event; the card's detail pane shows a growing trace of `(step, tool, argument, observation)` rows in real time via SSE.
5. When the agent completes the loop, the card transitions to `ANSWERED`. The detail pane shows: a short `answerText`, a `confidence` tag (HIGH / MEDIUM / LOW), the full tool-call trace, and the number of steps taken.
6. Within ~1 s of `ANSWERED`, the `evalStep` finishes. The card shows a **completeness score** chip (1–5) plus a one-line rationale — e.g. "Answer references the retrieved passage and the numeric result is consistent with the math trace."
7. The user can submit another question; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ResearchEndpoint` | `HttpEndpoint` | `/api/questions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `ResearchEntity`, `ResearchView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ResearchEntity` | `EventSourcedEntity` | Per-question lifecycle: submitted → running → answered / failed. Source of truth for the full tool-call trace. | `ResearchEndpoint`, `ResearchWorkflow` | `ResearchView` |
| `ResearchWorkflow` | `Workflow` | One workflow per question. Steps: `runStep` → `evalStep`. | started by `ResearchEndpoint` after entity created | `ResearchAgent`, `ResearchEntity` |
| `ResearchAgent` | `AutonomousAgent` | The one decision-making LLM. Runs the ReAct loop with `evaluate_math` and `search_wikipedia` tools; returns `ResearchAnswer`. | invoked by `ResearchWorkflow` | returns answer |
| `MathGuardrail` | supporting class | `before-tool-call` hook on `ResearchAgent`; validates `evaluate_math` arguments against the permitted grammar. | agent tool-call intercept | allows or rejects |
| `AnswerEvaluator` | supporting class | Deterministic rule-based scorer (no LLM); runs inside `evalStep`. | invoked by `ResearchWorkflow` | returns `EvalResult` |
| `ResearchView` | `View` | Read model: one row per question for the UI. | `ResearchEntity` events | `ResearchEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Question(
    String questionId,
    String text,
    String submittedBy,
    Instant submittedAt
) {}

record ToolCall(
    int step,
    ToolName tool,
    String argument,
    String observation
) {}
enum ToolName { EVALUATE_MATH, SEARCH_WIKIPEDIA }

record ResearchAnswer(
    String answerText,
    Confidence confidence,
    List<ToolCall> trace,
    int stepCount,
    Instant answeredAt
) {}
enum Confidence { HIGH, MEDIUM, LOW }

record EvalResult(
    int score,            // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record ResearchQuestion(
    String questionId,
    Optional<Question> question,
    Optional<ResearchAnswer> answer,
    Optional<EvalResult> eval,
    QuestionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum QuestionStatus {
    SUBMITTED, RUNNING, ANSWERED, EVALUATED, FAILED
}
```

Events on `ResearchEntity`: `QuestionSubmitted`, `RunStarted`, `ToolCallRecorded`, `AnswerRecorded`, `EvaluationScored`, `QuestionFailed`.

Every nullable lifecycle field on the `ResearchQuestion` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/questions` — body `{ text, submittedBy }` → `{ questionId }`.
- `GET /api/questions` — list all questions, newest-first.
- `GET /api/questions/{id}` — one question.
- `GET /api/questions/sse` — Server-Sent Events; one event per state transition and per tool-call step.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MathWikiReAct</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted questions (status pill + confidence badge + age) and a right pane with the selected question's detail — question text, real-time tool-call trace, answer text, confidence tag, and eval-score chip.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every `evaluate_math` tool call made by `ResearchAgent`. The `MathGuardrail` checks that the argument is a well-formed arithmetic expression containing only numeric literals, the operators `+`, `-`, `*`, `/`, `^`, `(`, `)`, and the functions `sqrt`, `abs`, `log`, `exp`. It rejects any argument that contains identifiers resolving outside the expression (no variable names, no function calls to unlisted identifiers, no string literals, no semicolons). On rejection, returns a structured error to the agent loop naming the forbidden construct; the agent may reformulate and retry within its iteration budget.

## 9. Agent prompts

- `ResearchAgent` → `prompts/research-agent.md`. The single decision-making LLM. System prompt instructs it to run the ReAct loop: given the question, reason about which tool to call, call it, observe the result, and repeat until the answer is ready.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits a numeric word-problem; the agent calls `evaluate_math` at least once; within 30 s a `ResearchAnswer` appears with `confidence = HIGH` and a numeric `answerText`.
2. **J2** — User submits a factual lookup; the agent calls `search_wikipedia` at least once; the `answerText` cites a phrase from the retrieved passage.
3. **J3** — The agent issues an `evaluate_math` call with a forbidden construct (mock LLM path); the `before-tool-call` guardrail rejects it; the agent retries with a conforming expression; the final answer is well-formed.
4. **J4** — A verdict whose `answerText` is one word and `trace` is empty receives an eval score of 1 with a clear rationale.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mathwikireact demonstrating the single-agent × research-intel cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-research-intel-react-math-wiki. Java package io.akka.samples.mathwikipediareact.
Akka 3.6.0. HTTP port 9355.

Components to wire (exactly):

- 1 AutonomousAgent ResearchAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/research-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_QUESTION).maxIterationsPerTask(8)).
  The agent has two tools wired:
    * evaluate_math(expression: String): String — evaluates the expression using the
      in-process MathEvaluator (restricted grammar; see G1). Returns the numeric result
      as a string.
    * search_wikipedia(query: String): String — looks up the query in the in-process
      WikipediaStub, returning the matching seeded passage or "(no passage found)".
  Output: ResearchAnswer{answerText: String, confidence: Confidence (HIGH/MEDIUM/LOW),
  trace: List<ToolCall>, stepCount: int, answeredAt: Instant}.
  The agent is configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml)
  registered via the agent's guardrail-configuration block. The guardrail applies only to
  evaluate_math calls; search_wikipedia calls pass through unchecked.

- 1 Workflow ResearchWorkflow per questionId with two steps:
  * runStep — emits RunStarted, then calls componentClient.forAutonomousAgent(
    ResearchAgent.class, "researcher-" + questionId).runSingleTask(
      TaskDef.instructions(question.text)
    ) — returns a taskId, then forTask(taskId).result(ANSWER_QUESTION) to fetch the answer.
    During execution the agent emits ToolCallRecorded events via a callback registered on
    the task context for each tool invocation. On success calls ResearchEntity.recordAnswer(answer).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(ResearchWorkflow::error).
  * evalStep — runs a deterministic rule-based AnswerEvaluator (NOT an LLM call) over the
    recorded answer: checks that answerText is more than one word, that trace is non-empty,
    that the answer references at least one observation from the trace, and that the confidence
    level is proportional to the number of corroborating tool calls. Emits
    EvaluationScored{score: 1-5, rationale: String}. WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ResearchEntity (one per questionId). State
  ResearchQuestion{questionId: String, question: Optional<Question>,
  answer: Optional<ResearchAnswer>, eval: Optional<EvalResult>,
  status: QuestionStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  QuestionStatus enum: SUBMITTED, RUNNING, ANSWERED, EVALUATED, FAILED.
  Events: QuestionSubmitted{question}, RunStarted{}, ToolCallRecorded{step, tool, argument,
  observation}, AnswerRecorded{answer}, EvaluationScored{eval}, QuestionFailed{reason}.
  Commands: submit, markRunning, recordToolCall, recordAnswer, recordEvaluation, fail,
  getQuestion. emptyState() returns ResearchQuestion.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 View ResearchView with row type QuestionRow (mirrors ResearchQuestion minus
  internal trace details — the view carries a truncated trace of up to 10 steps for the UI).
  Table updater consumes ResearchEntity events. ONE query getAllQuestions:
  SELECT * AS questions FROM research_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ResearchEndpoint at /api with POST /questions (body {text, submittedBy}; mints
    questionId; calls ResearchEntity.submit; starts ResearchWorkflow; returns {questionId}),
    GET /questions (list from getAllQuestions, sorted newest-first), GET /questions/{id}
    (one row), GET /questions/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ResearchTasks.java declaring one Task<R> constant: ANSWER_QUESTION = Task.name("Answer
  question").description("Run a ReAct loop with evaluate_math and search_wikipedia to
  answer the question").resultConformsTo(ResearchAnswer.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Question, ToolCall, ToolName, ResearchAnswer, Confidence, EvalResult,
  ResearchQuestion, QuestionStatus.

- MathGuardrail.java implementing the before-tool-call hook. On every evaluate_math call
  it tokenises the argument expression and checks: (1) only numeric literals, (2) operators
  in {+, -, *, /, ^, (, )}, (3) functions in {sqrt, abs, log, exp}, (4) no identifiers
  outside those functions, (5) no semicolons or statement separators. Returns
  Guardrail.allow() on success; Guardrail.reject(<structured-error>) naming the forbidden
  token on failure. Does not inspect search_wikipedia arguments.

- MathEvaluator.java — pure deterministic evaluator (no LLM, no shell exec). Implements
  the same grammar MathGuardrail checks. Inputs: a validated expression string. Outputs:
  the numeric result as a String (e.g. "42.0"). Throws EvaluationException if the
  expression violates the grammar at runtime (belt-and-suspenders beyond the guardrail).

- WikipediaStub.java — in-process lookup table. Reads
  src/main/resources/sample-events/wikipedia-passages.jsonl at startup; keys are
  lower-cased query strings; returns the longest-matching passage or "(no passage found)".

- AnswerEvaluator.java — pure deterministic logic (no LLM). Inputs: ResearchAnswer and the
  original Question. Outputs: EvalResult. Scoring rubric documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9355 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ResearchAgent.definition() binds
  the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-questions.jsonl with 5 seeded questions:
  (1) "How many meters is the circumference of the Earth?" (Wikipedia lookup)
  (2) "What is 17% of 850?" (pure math)
  (3) "What year was the Eiffel Tower completed, and how many years ago was that?" (mixed)
  (4) "What is the square root of 2025?" (pure math)
  (5) "How tall is Mount Everest in feet, given it is 8849 meters high?" (mixed)

- src/main/resources/sample-events/wikipedia-passages.jsonl with passages for:
  earth circumference, eiffel tower history, mount everest height, speed of light,
  diameter of the moon, population of Japan, distance to the moon — enough to exercise
  the agent's Wikipedia path on the seeded questions.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root reflecting domain research-intel, sector
  TO_BE_COMPLETED_BY_DEPLOYER, decisions.authority_level = inform-only (the agent's answer
  is informational; no autonomous action is taken), oversight.human_in_loop = false (answers
  are advisory; no human must approve before the answer is shown),
  failure.failure_modes including "hallucinated-fact", "math-overflow", "unsafe-code-exec",
  "incomplete-trace", "confidence-miscalibration"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/research-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: MathWikiReAct", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of question cards; right = selected-question detail with question text,
  real-time tool-call trace table, answer text, confidence badge, and eval-score chip).
  Browser title exactly: <title>Akka Sample: MathWikiReAct</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(questionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    answer-question.json — 6 ResearchAnswer entries:
      2 entries with confidence HIGH and a multi-step trace (3–4 steps) mixing both tools.
      2 entries with confidence MEDIUM and a 2-step trace.
      1 entry with confidence LOW and a 1-step trace (short answer).
      1 deliberately MALFORMED entry where the trace contains an evaluate_math step whose
      argument includes a semicolon — triggering the guardrail reject path. The mock
      selects this entry on the FIRST iteration of every 3rd question (modulo seed) so J3
      is reproducible.
- A MockModelProvider.seedFor(questionId) helper makes per-question selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ResearchAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ResearchTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (runStep
  90s, evalStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the ResearchQuestion row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ResearchTasks.java with ANSWER_QUESTION = Task.name(...).description(...)
  .resultConformsTo(ResearchAnswer.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9355 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ResearchAgent). The
  on-decision evaluator (AnswerEvaluator.java) is rule-based and does NOT make an LLM call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  bound specifically to the evaluate_math tool. search_wikipedia calls pass through
  without guardrail inspection.
- The question text is passed as the task's instruction string (TaskDef.instructions(text));
  no file attachment is used for this blueprint.
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
