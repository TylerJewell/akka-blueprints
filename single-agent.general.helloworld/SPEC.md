# Hello Agent — `single-agent.general.helloworld`

This file is the natural-language input for `/akka:specify`. Run `/akka:specify @SPEC.md` from inside this folder. Sections 1–11 together are the spec; Section 12 chains the rest of generation.

---

## 1. System name + one-line pitch

**Hello Agent.** A user types a question. A single `QuestionAnswerer` agent answers it and returns a typed `Answer{ text, confidence }`; an output check screens the answer before it is shown.

## 2. What this blueprint demonstrates

The single-agent coordination pattern at its smallest: one AutonomousAgent that accepts one ANSWER task and returns a typed result, persisted through an event-sourced entity and projected into a read model the UI streams. The governance pattern is one output guardrail — a before-agent-response check on the answer's confidence and content. There is no orchestration, no tool use, and no second agent.

## 3. User-facing flows

1. The user opens the App UI, types a question, and clicks **Ask**. The question appears in the list in `ASKED` state.
2. The agent runs the ANSWER task. Within roughly 5–30 seconds the question moves to `ANSWERED` and the answer text and a confidence score (0.0–1.0) appear.
3. If the answer fails the before-agent-response check (confidence below the threshold or empty/unusable content), the question moves to `BLOCKED` with a short reason and no answer text is shown.
4. The user can also ask through the API (`POST /api/ask`) and read the typed answer back from `GET /api/questions/{id}`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `QuestionAnswerer` | AutonomousAgent | Answers one question; returns `Answer{ text, confidence }` | `AnswerConsumer` | `QuestionEntity` |
| `AnswererTasks` | task definitions | Declares the `ANSWER` `Task<Answer>` constant | — | `QuestionAnswerer` |
| `QuestionEntity` | EventSourcedEntity | Holds each question's lifecycle | `AskEndpoint`, `AnswerConsumer` | `QuestionsView` |
| `QuestionsView` | View | Read model of all questions | `QuestionEntity` | `AskEndpoint` |
| `AnswerConsumer` | Consumer | On `QuestionAsked`, runs the ANSWER task and records the result | `QuestionEntity` | `QuestionAnswerer`, `QuestionEntity` |
| `AskEndpoint` | HttpEndpoint | `/api` surface — ask, list, single, SSE, metadata | UI / clients | `QuestionEntity`, `QuestionsView` |
| `AppEndpoint` | HttpEndpoint | Serves `/` redirect and `/app/*` static UI | browser | static-resources |
| `Bootstrap` | service-setup | Startup wiring | — | — |

Names are authoritative — `/akka:specify` uses them verbatim.

## 5. Data model

Full detail in `reference/data-model.md`. Records:

```java
public record Question(
  String id,
  String text,
  QuestionStatus status,
  Optional<Instant> askedAt,
  Optional<String>  answerText,
  Optional<Double>  confidence,
  Optional<Instant> answeredAt,
  Optional<String>  blockedReason,
  Optional<Instant> blockedAt
) {
  public static Question initial(String id) { /* all Optional.empty(), status ASKED-pending */ }
  public Question applyEvent(QuestionEvent e) { /* per-variant switch */ }
}

public enum QuestionStatus { ASKED, ANSWERED, BLOCKED }

public record Answer(String text, double confidence) {}
```

Every nullable lifecycle field is `Optional<T>` (Lesson 6). `QuestionEvent` is a sealed interface with three variants:

```java
sealed interface QuestionEvent permits QuestionAsked, AnswerRecorded, AnswerBlocked { Instant timestamp(); }
record QuestionAsked(String questionId, String text, Instant timestamp) implements QuestionEvent {}
record AnswerRecorded(String questionId, String text, double confidence, Instant timestamp) implements QuestionEvent {}
record AnswerBlocked(String questionId, String reason, Instant timestamp) implements QuestionEvent {}
```

## 6. API contract

Full schemas in `reference/api-contract.md`. Top-level surface:

```
POST /api/ask                     -> { questionId }
GET  /api/questions               -> { questions: [Question, ...] }
GET  /api/questions/{id}          -> Question
GET  /api/questions/sse           -> Server-Sent Events of Question
GET  /api/metadata/eval-matrix    -> text/yaml
GET  /api/metadata/risk-survey    -> text/yaml
GET  /api/metadata/readme         -> text/markdown
GET  /                            -> 302 /app/index.html
GET  /app/{*path}                 -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html`. No `ui/` folder, no npm build (Lesson 17). Browser title `<title>Akka Sample: Hello Agent</title>`. Five tabs, full description in `reference/ui-mockup.md`:

1. **Overview** — eyebrow + headline + Try it / How it works / Components / API contract cards.
2. **Architecture** — the four mermaid diagrams with the Lesson 24 CSS overrides (white state labels, `overflow:visible` edge labels, `transitionLabelColor #cccccc`).
3. **Risk Survey** — `risk-survey.yaml` in `matrix-card`/`matrix-row` style; `TO_BE_COMPLETED_BY_DEPLOYER` values muted.
4. **Eval Matrix** — `eval-matrix.yaml` rows with a colored mechanism pill in the label column.
5. **App UI** — ask box, live SSE list of questions, per-question answer text + confidence, or the block reason.

Tab switching matches by `data-tab` / `data-panel` attribute, never by NodeList index, and no hidden zombie panels remain in the DOM (Lesson 26).

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. One mechanism is wired:

- **G1 — guardrail, before-agent-response.** A check fires on the answer the `QuestionAnswerer` is about to return. If the confidence is below the configured threshold (default `0.4`) or the content is empty/unusable, the guardrail blocks the response; `AnswerConsumer` then records an `AnswerBlocked` event instead of `AnswerRecorded`. Enforcement is blocking — a blocked answer never reaches the user.

## 9. Agent prompts

- `QuestionAnswerer` → `prompts/question-answerer.md` — answers the question concisely and self-reports a calibrated confidence score.

## 10. Acceptance

Full journeys in `reference/user-journeys.md`. The three that must pass:

1. **Ask via UI → answered.** Type a question, see it reach `ANSWERED` with non-empty answer text and a confidence in `[0,1]`.
2. **Ask via API → typed answer.** `POST /api/ask`, then `GET /api/questions/{id}` returns the answer text and confidence.
3. **Low-quality answer → blocked.** A question whose answer fails the before-agent-response check lands in `BLOCKED` with a reason and no answer text.

## 11. Implementation directives

```
Create a sample named helloworld demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples.
Maven artifact helloworld. Java package io.akka.samples.helloworld.
Akka 3.6.0. HTTP port 9390.

Components to wire (exactly):
- 1 AutonomousAgent QuestionAnswerer: extends
  akka.javasdk.agent.autonomous.AutonomousAgent (never downgrade to Agent —
  Lesson 1). definition() sets .instructions(<prompts/question-answerer.md>)
  and .capability(TaskAcceptance.of(AnswererTasks.ANSWER).maxIterationsPerTask(2)).
  Returns a typed Answer{String text, double confidence}. A before-agent-response
  guardrail screens the Answer: if confidence < 0.4 or text is blank, block.
- 1 task-definitions companion AnswererTasks declaring
  ANSWER = Task.name("Answer question").description("Answer one user question")
  .resultConformsTo(Answer.class) (Lesson 7 — AutonomousAgent needs this file).
- 1 EventSourcedEntity QuestionEntity holding a Question record with id, text
  (non-optional, set at creation), QuestionStatus enum, and Optional lifecycle
  fields (askedAt, answerText, confidence, answeredAt, blockedReason, blockedAt).
  Events: QuestionAsked, AnswerRecorded, AnswerBlocked. Commands: ask(text),
  recordAnswer(text, confidence), blockAnswer(reason), getQuestion.
  emptyState() returns Question.initial("") with NO commandContext() reference
  (Lesson 3).
- 1 View QuestionsView with row type Question, table updater consuming
  QuestionEntity events. ONE query: getAllQuestions SELECT * AS questions FROM
  questions_view. No WHERE on the status enum (Akka cannot auto-index enum
  columns — Lesson 2); filter client-side in callers.
- 1 Consumer AnswerConsumer subscribed to QuestionEntity events; on
  QuestionAsked it calls forAutonomousAgent(QuestionAnswerer.class,
  "answer-"+questionId).runSingleTask(ANSWER.instructions(question text)), then
  forTask(taskId).result(ANSWER). On a returned Answer it calls
  QuestionEntity.recordAnswer; if the guardrail blocked the response it calls
  QuestionEntity.blockAnswer(reason).
- 1 HttpEndpoint AskEndpoint at /api: ask (POST), questions list (client-side
  filter from getAllQuestions), single question, SSE stream
  (serverSentEventsForView over QuestionsView), and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/.
- 1 HttpEndpoint AppEndpoint serving / -> 302 /app/index.html and /app/* ->
  static-resources/*.
- 1 service-setup Bootstrap.

Companion files:
- Answer(String text, double confidence).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9390 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify model names are current before locking
  (Lesson 8).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the single control G1 and a
  matching simplified_view entry. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions,
  data.types, capability.*, model.*, subjects.children; marking deployer
  fields TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/, no npm). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture
  (four mermaid diagrams with Lesson 24 CSS overrides), Risk Survey, Eval
  Matrix (colored mechanism pill in the label column), App UI (ask box, SSE
  list, answer text + confidence, block reason). Match the governance.html
  visual style (dark / yellow accent / Instrument Sans / dot-grid). Tab
  switching by data-tab / data-panel attribute, never NodeList index; delete
  removed panels rather than display:none (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random
  outputs (QuestionAnswerer -> Answer; see
  src/main/resources/mock-responses/question-answerer.json with 4-6 entries,
  each an Answer{text, confidence} including at least one low-confidence entry
  so the BLOCKED path is exercised). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured
  reference if it does not resolve at runtime; never echoes captured key.

Mock LLM provider — per-agent mock-response shapes:
- QuestionAnswerer -> Answer{ text:String, confidence:double }. Provide 4-6
  entries in mock-responses/question-answerer.json; at least one with
  confidence < 0.4 to exercise the before-agent-response block path.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: every agent-calling step/consumer uses a timeout long enough for
  an LLM call (>= 60s); never rely on the 5s default.
- Lesson 6: Optional<T> for every nullable lifecycle field on the View row.
- Lesson 7: AutonomousAgent requires the AnswererTasks companion.
- Lesson 8: verify model names current before locking application.conf.
- Lesson 9: run command is "/akka:build" (Claude Code slash command), never
  "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9390 in application.conf.
- Lesson 11: never render any source-platform metadata in user-facing surfaces.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names anywhere user-facing.
- Lesson 24: mermaid state-diagram label CSS overrides + theme variables.
- Lesson 25: five-option key sourcing; never write the key to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; no zombie panels.
```

## 12. Post-scaffolding workflow

When `/akka:specify` finishes scaffolding the spec into `specs/features/`, **do not stop and wait for the user**. Continue automatically:

1. Run `/akka:plan` — pick the most conservative option for anything ambiguous, consistent with Sections 4 and the PLAN.
2. Run `/akka:tasks` — same default-everything policy.
3. Run `/akka:implement` — execute every task. If a task fails, capture it, continue with the next, and summarise all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service running, output just the listening URL and a one-line summary of any failures. **Stop earlier only** for a hard error you cannot work around without the user: an unresolved API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
