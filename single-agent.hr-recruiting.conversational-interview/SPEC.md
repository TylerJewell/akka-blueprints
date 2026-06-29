# SPEC — conversational-interview

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ConversationalInterview.
**One-line pitch:** A coordinator starts an interview session for a role; one AI agent reads the role definition and conversation history (passed as task attachments, never as inline prompt text) and produces the next question to ask — adapting based on prior answers until the session is complete.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the hr-recruiting domain. One `InterviewConductorAgent` (AutonomousAgent) carries all question-generation decisions; the surrounding components prepare its input and govern its output. Two governance mechanisms are wired around the agent:

- A **special-category sanitizer** runs inside a Consumer between each submitted answer and the agent call — so the model never processes protected-category attributes (race, religion, pregnancy status, disability, political opinion, and similar GDPR/EEOC special categories).
- A **before-agent-response guardrail** validates every question the agent intends to send before it is shown to the candidate: no prohibited topics, no leading language that pressures a protected disclosure, each question mapped to a declared competency in the role definition. A rejected question triggers a retry within the same task.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks bracket the one decision-making LLM call without adding a second agent.

## 3. User-facing flows

The user opens the App UI tab.

1. The coordinator picks a **role** from a dropdown (Software Engineer, Product Manager, Customer Success) or pastes a custom role definition (JSON with competencies list + interview depth).
2. The coordinator enters a **candidate identifier** (anonymised handle) and clicks **Start session**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
3. The session card appears in the live list in `OPEN` state. Within ~1 s, the `conductTurnStep` fires. The first question appears in the right pane — the agent generated it from the role definition alone (no prior answers yet).
4. The coordinator reads the question to the candidate and enters the candidate's answer into the **Answer** textarea, then clicks **Submit answer**. The UI POSTs to `/api/sessions/{id}/answers`.
5. The card transitions to `ANSWER_RECEIVED`. Within ~1 s it transitions to `ANSWER_SCREENED` — the sanitizer's screening result is visible (categories detected, if any, and the redacted text).
6. Within ~10–30 s, the `conductTurnStep` completes. The next question appears. Steps 4–6 repeat until the agent signals the interview is complete.
7. When the agent returns a turn with `sessionComplete: true`, the session transitions to `COMPLETED`. The full transcript appears — each turn's question, the candidate's answer, and the screening result per turn.
8. The coordinator can start another session; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — start, submit-answer, list, get, SSE; serves `/api/metadata/*`. | — | `InterviewSessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `InterviewSessionEntity` | `EventSourcedEntity` | Per-session lifecycle: open → answer-received → answer-screened → conducting → completed / failed. Source of truth. | `SessionEndpoint`, `AnswerSanitizer`, `InterviewWorkflow` | `SessionView` |
| `AnswerSanitizer` | `Consumer` | Subscribes to `AnswerSubmitted` events; screens for protected-category attributes; calls `InterviewSessionEntity.attachScreenedAnswer`. | `InterviewSessionEntity` events | `InterviewSessionEntity` |
| `InterviewWorkflow` | `Workflow` | One workflow per session. Steps: `awaitScreenedStep` → `conductTurnStep` → check-complete → loop or `completeStep`. | started by `AnswerSanitizer` once answer is screened; also started directly after session open | `InterviewConductorAgent`, `InterviewSessionEntity` |
| `InterviewConductorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the role definition and conversation history as task attachments; returns `InterviewTurn`. | invoked by `InterviewWorkflow` | returns next question |
| `SessionView` | `View` | Read model: one row per session for the UI. | `InterviewSessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Competency(String competencyId, String name, String description) {}

record RoleDefinition(
    String roleId,
    String roleTitle,
    List<Competency> competencies,
    int targetTurnCount,          // how many Q&A rounds before session complete
    String gradingNote            // optional guidance for the conductor
) {}

record SessionRequest(
    String sessionId,
    String candidateHandle,       // anonymised — never a real name
    RoleDefinition role,
    String startedBy,
    Instant startedAt
) {}

record SubmittedAnswer(
    String turnIndex,
    String rawAnswer,
    Instant answeredAt
) {}

record ScreenedAnswer(
    String turnIndex,
    String redactedAnswer,
    List<String> protectedCategoriesFound
) {}

record InterviewTurn(
    String turnIndex,
    String question,
    String competencyId,          // which competency this question targets
    boolean sessionComplete,
    Instant generatedAt
) {}

record TurnRecord(
    String turnIndex,
    InterviewTurn turn,
    Optional<SubmittedAnswer> submittedAnswer,
    Optional<ScreenedAnswer> screenedAnswer
) {}

record InterviewSession(
    String sessionId,
    Optional<SessionRequest> request,
    List<TurnRecord> turns,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> completedAt
) {}

enum SessionStatus {
    OPEN, ANSWER_RECEIVED, ANSWER_SCREENED, CONDUCTING, COMPLETED, FAILED
}
```

Events on `InterviewSessionEntity`: `SessionStarted`, `AnswerSubmitted`, `AnswerScreened`, `TurnConducted`, `SessionCompleted`, `SessionFailed`.

Every nullable lifecycle field on `InterviewSession` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ candidateHandle, role: RoleDefinition, startedBy }` → `{ sessionId }`.
- `POST /api/sessions/{id}/answers` — body `{ turnIndex, rawAnswer }` → `202 accepted`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ConversationalInterview</title>`.

The App UI tab is a two-column layout: a left rail with the live list of interview sessions (status pill + role badge + age) and a right pane with the selected session's detail — role competencies, the turn-by-turn transcript (question, screening result, candidate answer), and a completion summary when the session reaches `COMPLETED`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — Special-category sanitizer** (`special-category`, applied inside `AnswerSanitizer` Consumer): screens each submitted answer for GDPR/EEOC protected-category attributes — race, religion, ethnic origin, political opinion, trade-union membership, health, sexual orientation, pregnancy status — before any LLM call. Records which categories were detected and replaces the flagged spans with typed redaction tokens (e.g., `[REDACTED-HEALTH]`, `[REDACTED-RELIGION]`). The raw answer is preserved on the entity for audit; only the screened form reaches the agent.
- **G1 — before-agent-response guardrail**: runs on every turn of `InterviewConductorAgent`. Asserts the candidate question is not blank, maps to a declared `competencyId` in the role definition, contains no prohibited-topic markers (pregnancy, family planning, religious practice, disability accommodation, protected financial questions), and does not lead the candidate toward a protected disclosure. On failure, returns a structured rejection code to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `InterviewConductorAgent` → `prompts/interview-conductor.md`. The single decision-making LLM. System prompt instructs it to read the role definition attachment, review the conversation-history attachment, and return one `InterviewTurn` with the next question (or the final question with `sessionComplete: true`).

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Coordinator starts a Software Engineer session; within 30 s per turn the agent produces a question; session reaches `COMPLETED` with one `TurnRecord` per conducted turn and a transcript.
2. **J2** — A candidate answer mentions a health condition; the `AnswerSanitizer` redacts it; the agent's subsequent question does not reference the health content; the UI shows the redacted form in the transcript.
3. **J3** — The agent drafts a question containing prohibited language; the `before-agent-response` guardrail rejects it; the second iteration produces a compliant question; the UI never displays the rejected draft.
4. **J4** — A session's entity log shows the raw answer with the protected-category span; the `SessionView` row and the UI show only the redacted form.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named conversational-interview demonstrating the single-agent × hr-recruiting cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-hr-recruiting-conversational-interview. Java package
io.akka.samples.conversationalinterviewsviaaiform. Akka 3.6.0. HTTP port 9427.

Components to wire (exactly):

- 1 AutonomousAgent InterviewConductorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/interview-conductor.md>) and
  .capability(TaskAcceptance.of(CONDUCT_TURN).maxIterationsPerTask(3)). The task receives
  the role definition as a task ATTACHMENT named "role.json" and the conversation history as
  a task ATTACHMENT named "history.json" (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  InterviewTurn{turnIndex: String, question: String, competencyId: String,
  sessionComplete: boolean, generatedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow InterviewWorkflow per sessionId with three phases:
  * On session open (no prior answers): go directly to conductTurnStep.
  * awaitScreenedStep — polls InterviewSessionEntity.getSession every 1s; on the latest
    SubmittedAnswer having a matching ScreenedAnswer advances to conductTurnStep.
    WorkflowSettings.stepTimeout 15s.
  * conductTurnStep — emits TurnConducting, then calls componentClient.forAutonomousAgent(
    InterviewConductorAgent.class, "conductor-" + sessionId).runSingleTask(
      TaskDef.instructions(formatTurnPrompt(session))
        .attachment("role.json", serialize(session.request.role).getBytes())
        .attachment("history.json", serialize(session.turns).getBytes())
    ) — returns a taskId, then forTask(taskId).result(CONDUCT_TURN) to fetch the turn.
    On success calls InterviewSessionEntity.recordTurn(turn). If turn.sessionComplete is
    true, transitions to completeStep; otherwise transitions to awaitAnswerStep (waiting
    for the next candidate answer, which re-triggers awaitScreenedStep).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(InterviewWorkflow::error).
  * completeStep — calls InterviewSessionEntity.complete(). WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity InterviewSessionEntity (one per sessionId). State InterviewSession{
  sessionId: String, request: Optional<SessionRequest>, turns: List<TurnRecord>,
  status: SessionStatus, createdAt: Instant, completedAt: Optional<Instant>}.
  SessionStatus enum: OPEN, ANSWER_RECEIVED, ANSWER_SCREENED, CONDUCTING, COMPLETED, FAILED.
  Events: SessionStarted{request}, AnswerSubmitted{answer: SubmittedAnswer},
  AnswerScreened{screened: ScreenedAnswer}, TurnConducted{turn: InterviewTurn},
  SessionCompleted{completedAt}, SessionFailed{reason}. Commands: start, submitAnswer,
  attachScreenedAnswer, recordTurn, complete, fail, getSession. emptyState() returns
  InterviewSession.initial("") with all Optional fields as Optional.empty() and
  turns = Collections.emptyList() (Lesson 3). Every Optional<T> field uses Optional.empty()
  in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer AnswerSanitizer subscribed to InterviewSessionEntity events; on AnswerSubmitted
  runs a keyword+heuristic screening pipeline over rawAnswer detecting the following
  protected categories: race/ethnicity (enum marker), religion, political-opinion,
  health/disability, pregnancy/maternity, sexual-orientation, trade-union-membership,
  financial-status-proxy. Replaces flagged spans with typed redaction tokens
  ([REDACTED-RACE], [REDACTED-RELIGION], [REDACTED-HEALTH], [REDACTED-PREGNANCY],
  [REDACTED-POLITICAL], [REDACTED-UNION], [REDACTED-SEXUAL-ORIENTATION],
  [REDACTED-FINANCIAL]). Builds ScreenedAnswer{turnIndex, redactedAnswer,
  protectedCategoriesFound}. Calls InterviewSessionEntity.attachScreenedAnswer(screened).

- 1 View SessionView with row type SessionRow (mirrors InterviewSession minus
  TurnRecord.submittedAnswer.rawAnswer — the audit entity keeps the raw; the view holds
  the screened form for the UI). Table updater consumes InterviewSessionEntity events.
  ONE query getAllSessions: SELECT * AS sessions FROM session_view. No WHERE status filter
  — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {candidateHandle, role: {roleId,
    roleTitle, competencies: [{competencyId, name, description}], targetTurnCount,
    gradingNote}, startedBy}; mints sessionId; calls InterviewSessionEntity.start; returns
    {sessionId}), POST /sessions/{id}/answers (body {turnIndex, rawAnswer}; calls
    InterviewSessionEntity.submitAnswer; returns 202), GET /sessions (list from
    getAllSessions, sorted newest-first), GET /sessions/{id} (one row), GET /sessions/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- InterviewTasks.java declaring one Task<R> constant: CONDUCT_TURN = Task.name("Conduct
  interview turn").description("Read the role definition and conversation history and
  produce the next InterviewTurn question").resultConformsTo(InterviewTurn.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Competency, RoleDefinition, SessionRequest, SubmittedAnswer,
  ScreenedAnswer, InterviewTurn, TurnRecord, InterviewSession, SessionStatus.

- QuestionGuardrail.java implementing the before-agent-response hook. Reads the candidate
  InterviewTurn from the LLM response, runs the four checks listed in eval-matrix.yaml G1
  (non-blank question, competencyId exists in role, no prohibited topics, no
  protected-disclosure leading language), and either passes the response through or returns
  Guardrail.reject(<structured-error>) to force the agent loop to retry.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9427 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The InterviewConductorAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/role-definitions.jsonl with 3 seeded role definitions:
  a 5-competency Software Engineer guide, a 6-competency Product Manager guide, and a
  4-competency Customer Success guide. Each has targetTurnCount matching its competency
  count.

- src/main/resources/sample-events/seed-histories.jsonl with 3 paired partial conversation
  histories for the seeded roles — one after the first answer in each role, suitable for
  demonstrating session resume. Each history includes one answer containing a natural
  protected-category reference so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, G1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.special_category_personal_data
  = true, special_category_handled_by_sanitizer_before_llm = true,
  decisions.authority_level = assist-only (the agent generates questions; the coordinator
  and any human hiring process make all candidate decisions), oversight.human_in_loop = true
  (a coordinator reads every question before it is put to the candidate),
  failure.failure_modes including "biased-question-generation", "protected-disclosure-elicitation",
  "special-category-leakage-via-llm", "question-off-competency"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/interview-conductor.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Conversational Interviews via AI Form",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session detail with role competencies,
  per-turn transcript showing question + screening result + candidate answer, and completion
  summary). Browser title exactly:
  <title>Akka Sample: ConversationalInterview</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    conduct-turn.json — 10 InterviewTurn entries covering the three seeded roles.
      Each entry has a non-blank question, a competencyId that maps to a competency in the
      matched role, and sessionComplete: false for all but the last entry per role (the
      final entry per role has sessionComplete: true). Plus 2 deliberately MALFORMED entries
      (one with a blank question; one with a competencyId that does not exist in the role)
      — the guardrail blocks both, exercising the retry path. The mock should select a
      malformed entry on the FIRST iteration of every 4th session (modulo seed) so J3 is
      reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. InterviewConductorAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion
  InterviewTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout
  (conductTurnStep 60s, awaitScreenedStep 15s, completeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on InterviewSession is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: InterviewTasks.java with CONDUCT_TURN = Task.name(...).description(...)
  .resultConformsTo(InterviewTurn.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9427 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (InterviewConductorAgent). The special-category screen is rule-based (AnswerSanitizer)
  and does NOT make an LLM call — keeping the pattern's "one agent" promise honest.
- The role definition and conversation history are passed as Task ATTACHMENTS, never inlined
  into the agent's instructions. Verify the generated conductTurnStep uses
  TaskDef.attachment(...) and not string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference for how the hook is registered.
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
