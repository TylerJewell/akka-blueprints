# SPEC — with Signals & Queries

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** SignalDrivenChat.
**One-line pitch:** A user opens a chat session backed by a long-running Workflow; signals inject additional user prompts mid-flight and queries return the session's current state, enabling interactive multi-turn conversations without restarting the workflow.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern extended with Workflow signals and queries in the general domain. One `ChatAgent` (AutonomousAgent) handles every turn; the surrounding components manage the session lifecycle, relay signals, and safeguard what the agent receives. Two governance mechanisms are wired:

- A **before-agent-invocation guardrail** validates every inbound signal payload — blank prompts, oversized inputs, and payloads containing disallowed content are rejected before the agent sees them.
- An **application-level HITL** gate: an operator can signal a running session into `PAUSED` status, inject a corrective prompt, and resume — giving a human explicit control over the agent's next context without terminating the workflow.

The blueprint shows that signals and queries are not ad-hoc HTTP calls — they are first-class workflow primitives that carry governance just as strongly as the initial request.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a session name and an initial prompt into the **New session** panel and clicks **Start session**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
2. The session card appears in the live list in `STARTING` state. Within ~1 s it transitions to `ACTIVE` — the workflow's `processTurnStep` fires, the agent runs, and the first reply appears in the session detail.
3. The user types a follow-up prompt into the **Add turn** input field on the detail pane and clicks **Send**. The UI sends `PUT /api/sessions/{id}/signal/turn` with the new prompt body.
4. The `SignalValidator` guardrail checks the payload. If valid, the workflow's `processTurnStep` fires again with the full accumulated conversation; a new reply appears.
5. At any point, the user (or an operator) clicks **Pause** to halt turn processing. The UI sends `PUT /api/sessions/{id}/signal/pause`. The session transitions to `PAUSED`; the workflow suspends. A text field appears for an operator-supplied corrective prompt.
6. The operator types a note and clicks **Resume**. The UI sends `PUT /api/sessions/{id}/signal/resume` with the optional corrective note body. The workflow resumes with the note prepended to the next turn's context.
7. The user clicks **Query state** at any time. The UI calls `GET /api/sessions/{id}/query` — a read that does not trigger an agent turn. The detail pane refreshes immediately with the workflow's in-memory state.
8. The session ends when the user clicks **Close**. The workflow's `closeStep` writes a `SessionClosed` event and transitions to `CLOSED`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/sessions/*` — start, list, get, signal, query, SSE; serves `/api/metadata/*`. | — | `ChatSessionEntity`, `SessionView`, `ChatSessionWorkflow` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ChatSessionEntity` | `EventSourcedEntity` | Per-session event log: tracks all turns and status transitions. Source of truth. | `ChatEndpoint`, `ChatSessionWorkflow` | `SessionView` |
| `ChatSessionWorkflow` | `Workflow` | One workflow per session. Steps: `processTurnStep` → waits for next signal → loops or `closeStep`. Accepts `addTurnSignal`, `pauseSignal`, `resumeSignal`, `closeSignal`. Exposes `getSessionQuery`. | started by `ChatEndpoint` | `ChatAgent`, `ChatSessionEntity` |
| `SignalValidator` | supporting class | Validates `AddTurnSignal` payloads before `ChatSessionWorkflow` passes them to the agent. Registered as the before-agent-invocation guardrail on `ChatAgent`. | invoked by `ChatSessionWorkflow` before each task | `ChatAgent` |
| `ChatAgent` | `AutonomousAgent` | The single decision-making LLM. Receives the accumulated conversation context and the new prompt; returns `AgentReply`. | invoked by `ChatSessionWorkflow` | returns reply |
| `SessionView` | `View` | Read model: one row per session for the UI. | `ChatSessionEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TurnRecord(
    String turnId,
    String prompt,
    String reply,
    TurnSource source,       // USER, OPERATOR, SYSTEM
    Instant sentAt,
    Instant repliedAt
) {}
enum TurnSource { USER, OPERATOR, SYSTEM }

record AddTurnSignal(
    String prompt,
    TurnSource source        // defaults to USER from the API
) {}

record PauseSignal(String reason) {}

record ResumeSignal(
    String correctiveNote    // prepended to next turn context; may be empty
) {}

record AgentReply(
    String replyText,
    Instant generatedAt
) {}

record SessionState(
    String sessionId,
    String sessionName,
    List<TurnRecord> turns,
    SessionStatus status,
    Optional<String> pauseReason,
    Instant startedAt,
    Optional<Instant> closedAt
) {}
enum SessionStatus { STARTING, ACTIVE, PAUSED, CLOSING, CLOSED, FAILED }

record SessionRow(
    String sessionId,
    String sessionName,
    int turnCount,
    SessionStatus status,
    Optional<String> lastReply,
    Instant startedAt,
    Optional<Instant> closedAt
) {}
```

Events on `ChatSessionEntity`: `SessionOpened`, `TurnAdded`, `SessionPaused`, `SessionResumed`, `SessionClosed`, `SessionFailed`.

Every nullable lifecycle field on `SessionState` and `SessionRow` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ sessionName, initialPrompt, submittedBy }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session (full turn log).
- `PUT /api/sessions/{id}/signal/turn` — body `{ prompt, source }` (inject a turn signal). Returns `204`.
- `PUT /api/sessions/{id}/signal/pause` — body `{ reason }`. Returns `204`.
- `PUT /api/sessions/{id}/signal/resume` — body `{ correctiveNote }`. Returns `204`.
- `PUT /api/sessions/{id}/signal/close` — no body. Returns `204`.
- `GET /api/sessions/{id}/query` — returns current `SessionState` from the workflow query without triggering a turn. Returns `200 SessionState`.
- `GET /api/sessions/sse` — Server-Sent Events; one event per turn completion or status transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: with Signals &amp; Queries</title>`.

The App UI tab is a two-column layout: a left rail with the live list of sessions (status pill + turn count + age) and a right pane with the selected session's detail — conversation thread, signal buttons (Pause / Resume / Close), corrective-note field, and a Query State button.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — application-level HITL** (`hitl`, `application`): the `pauseSignal` transitions the workflow and entity to `PAUSED`, suspending turn processing. An operator then supplies a `resumeSignal` with an optional `correctiveNote` that is prepended to the next turn's context. This is explicit human control over agent context mid-flight — not a post-hoc review.
- **G1 — before-agent-invocation guardrail** (`guardrail`, `before-agent-invocation`): `SignalValidator` runs on every `addTurnSignal` payload before `ChatSessionWorkflow` constructs the task for `ChatAgent`. Checks: (1) prompt is non-empty and ≤ 8000 characters, (2) prompt does not contain a disallowed-content marker (a configurable blocked-phrase list in `application.conf`), (3) source is a valid `TurnSource` enum value. Rejected signals are returned as a `400` response; the workflow does not advance.

## 9. Agent prompts

- `ChatAgent` → `prompts/chat-agent.md`. The single decision-making LLM. System prompt instructs it to maintain a coherent conversation, observe the accumulated turn history, and return a plain-text `AgentReply`. When an operator note is present, the agent acknowledges it before continuing.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User starts a session with an initial prompt; within 30 s the first agent reply appears in the UI turn list and the session shows `ACTIVE`.
2. **J2** — User injects a follow-up signal; the workflow processes the accumulated context; a second turn appears. The session's turn count increments to 2.
3. **J3** — An operator signals `pause`; the session enters `PAUSED`; no new agent turns fire. The operator sends `resume` with a corrective note; the session re-enters `ACTIVE` and the next agent reply incorporates the note.
4. **J4** — A signal with a blank `prompt` string is rejected with `400`; the session's turn count does not change; the guardrail rejection is visible in the service log.
5. **J5** — `GET /api/sessions/{id}/query` returns the full `SessionState` within 100 ms without adding a turn to the entity log.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named signal-driven-chat demonstrating the single-agent × general cell
with Workflow signals and queries. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact single-agent-general-akka-signal-driven-chat.
Java package io.akka.samples.withsignalsqueries. Akka 3.6.0. HTTP port 9231.

Components to wire (exactly):

- 1 AutonomousAgent ChatAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/chat-agent.md>) and
  .capability(TaskAcceptance.of(PROCESS_TURN).maxIterationsPerTask(3)). The task
  receives the full conversation context (all prior turns formatted as a numbered list)
  plus the new prompt in the task's instructions field. Output: AgentReply{replyText:
  String, generatedAt: Instant}. The agent is configured with a before-agent-invocation
  guardrail (see G1 in eval-matrix.yaml) registered via the agent's guardrail-
  configuration block. Rejected invocations cause the workflow to fail the current step
  and record a SessionFailed event on the entity.

- 1 Workflow ChatSessionWorkflow per sessionId. The workflow manages a turn-processing
  loop rather than a fixed linear sequence:
  * initStep — writes SessionOpened to ChatSessionEntity; advances to processTurnStep.
    WorkflowSettings.stepTimeout 5s.
  * processTurnStep — emits TurnAdded (partial, reply pending), builds the task with
    formatted conversation history as instructions and the new prompt appended, calls
    componentClient.forAutonomousAgent(ChatAgent.class, "chat-" + sessionId)
    .runSingleTask(TaskDef.instructions(formattedContext)), receives AgentReply, calls
    ChatSessionEntity.recordReply(turnId, reply). Then waits for the next signal.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ChatSessionWorkflow::failStep).
  * pausedStep — entered on pauseSignal. The workflow suspends here; no agent calls
    fire until resumeSignal arrives. WorkflowSettings.stepTimeout 300s (5-minute
    operator window).
  * closeStep — writes SessionClosed to ChatSessionEntity; the workflow terminates.
    WorkflowSettings.stepTimeout 5s.
  * failStep — writes SessionFailed to ChatSessionEntity; the workflow terminates.
    WorkflowSettings.stepTimeout 5s.

  Signals:
  * addTurnSignal(AddTurnSignal) — accepted in ACTIVE state. Passes payload through
    SignalValidator; on pass, stores signal in workflow state and transitions to
    processTurnStep. On reject, returns error without advancing.
  * pauseSignal(PauseSignal) — accepted in ACTIVE state. Transitions to pausedStep;
    records SessionPaused on entity.
  * resumeSignal(ResumeSignal) — accepted in PAUSED state. Prepends correctiveNote to
    next turn's context if non-empty; transitions to processTurnStep; records
    SessionResumed on entity.
  * closeSignal() — accepted in ACTIVE or PAUSED state. Transitions to closeStep.

  Query:
  * getSessionQuery() — returns the workflow's in-memory SessionState WITHOUT writing
    any event or advancing any step. Available in all non-terminal states.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ChatSessionEntity (one per sessionId). State SessionState{
  sessionId: String, sessionName: String, turns: List<TurnRecord>,
  status: SessionStatus, pauseReason: Optional<String>, startedAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus enum: STARTING, ACTIVE, PAUSED,
  CLOSING, CLOSED, FAILED. Events: SessionOpened{sessionId, sessionName, startedAt},
  TurnAdded{turnId, prompt, source, sentAt}, TurnReplied{turnId, replyText, repliedAt},
  SessionPaused{reason}, SessionResumed{correctiveNote}, SessionClosed{closedAt},
  SessionFailed{reason}. Commands: openSession, addTurn, recordReply, pauseSession,
  resumeSession, closeSession, failSession, getSession. emptyState() returns
  SessionState.initial("") with all Optional fields as Optional.empty() and
  status = STARTING. emptyState() never references commandContext() (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 supporting class SignalValidator implementing the before-agent-invocation hook.
  On each AddTurnSignal it (1) rejects if prompt is blank or whitespace-only,
  (2) rejects if prompt length exceeds 8000 characters, (3) rejects if prompt contains
  any token from a blocked-phrase list read from application.conf
  (akka.javasdk.agent.blocked-phrases), (4) rejects if source is not a valid
  TurnSource enum value. On any rejection returns a structured error naming the check
  that failed; the calling workflow handles this as a failed step.

- 1 View SessionView with row type SessionRow (sessionId, sessionName, turnCount,
  status, lastReply: Optional<String>, startedAt, closedAt: Optional<Instant>).
  Table updater consumes ChatSessionEntity events. ONE query getAllSessions:
  SELECT * AS sessions FROM session_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ChatEndpoint at /api with POST /sessions (body {sessionName, initialPrompt,
    submittedBy}; mints sessionId; calls ChatSessionEntity.openSession; starts
    ChatSessionWorkflow; returns {sessionId}), GET /sessions (list from
    getAllSessions, newest-first), GET /sessions/{id} (full session from entity),
    PUT /sessions/{id}/signal/turn (relays AddTurnSignal to workflow), PUT
    /sessions/{id}/signal/pause (relays PauseSignal), PUT /sessions/{id}/signal/
    resume (relays ResumeSignal), PUT /sessions/{id}/signal/close (relays
    closeSignal), GET /sessions/{id}/query (calls workflow.getSessionQuery()),
    GET /sessions/sse (Server-Sent Events from view stream-updates), and three
    /api/metadata/* endpoints serving YAML/MD from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- ChatTasks.java declaring one Task<R> constant: PROCESS_TURN = Task
  .name("Process turn").description("Continue the conversation given the accumulated
  context and the new prompt, returning an AgentReply").resultConformsTo(AgentReply
  .class). DO NOT skip this — the AutonomousAgent requires its companion Tasks class
  (Lesson 7).

- Domain records TurnRecord, AddTurnSignal, PauseSignal, ResumeSignal, AgentReply,
  SessionState, SessionRow, TurnSource, SessionStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9231,
  akka.javasdk.agent.blocked-phrases = [] (empty list; deployer-configurable), and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/sessions.jsonl with 3 seeded session scenarios:
  an open Q&A session (3 initial turns), a code-review assistant session (2 turns),
  and a meeting-notes summariser session (2 turns). Each includes a pause+resume
  turn to exercise the HITL path.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (H1, G1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.
  No regulation_anchors.

- risk-survey.yaml at the project root with operations.agent_pattern = single-agent,
  oversight.human_in_loop = true (operator can pause and inject), decisions.
  authority_level = TO_BE_COMPLETED_BY_DEPLOYER (general-purpose assistant; deployer
  sets this), data.data_classes.pii = TO_BE_COMPLETED_BY_DEPLOYER, failure.
  failure_modes including "off-topic-reply", "prompt-injection-via-signal",
  "runaway-session-cost", "guardrail-bypass"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/chat-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: with Signals & Queries",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no
  ui/, no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column
  layout (left = live list of sessions with status pill, turn count, age; right =
  selected-session detail with conversation thread, signal buttons, operator panel,
  Query State button). Browser title exactly:
  <title>Akka Sample: with Signals &amp; Queries</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM via the
        MCP tool's environment parameter.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured
  key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id. Each branch reads src/main/resources/mock-
  responses/process-turn.json, picks one entry pseudo-randomly per call (seedFor(
  sessionId, turnIndex)), and deserialises into AgentReply.
- Per-task mock-response shapes:
    process-turn.json — 10 AgentReply entries spanning question-answering, code-
    review commentary, and meeting-notes summary styles. 2 entries are deliberately
    empty-text (empty replyText string) to exercise the guardrail path. The mock
    selects an empty-text entry on the first turn of every 4th session so J4 is
    reproducible.
- MockModelProvider.seedFor(sessionId, turnIndex) makes selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChatAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChatTasks.java MUST
  exist.
- Lesson 4: every workflow step has an explicit stepTimeout (initStep 5s,
  processTurnStep 60s, pausedStep 300s, closeStep 5s, failStep 5s).
- Lesson 6: every nullable lifecycle field is Optional<T>. View updater wraps with
  Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: ChatTasks.java with PROCESS_TURN = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9231 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative or UI copy.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block. Without these, state names render
  black-on-black and arrow labels clip.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (ChatAgent). SignalValidator
  is a supporting class, not a second agent.
- The workflow query (getSessionQuery) must NOT emit an event or advance a step.
  It reads only the workflow's in-memory state.
- The before-agent-invocation guardrail is wired on the agent's guardrail-
  configuration block, not as an ad-hoc check in the workflow step before calling
  the agent.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
  Per Lesson 25, /akka:specify handles the key during generation.
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
