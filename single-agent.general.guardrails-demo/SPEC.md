# SPEC — guardrails-demo

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GuardrailsDemo.
**One-line pitch:** A user sends a free-text message to a single conversational agent; a `before-llm-call` topic-policy guardrail intercepts disallowed topics before the model is invoked, and a `before-agent-response` content-policy guardrail validates the reply before it leaves the agent loop.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `ConversationAgent` (AutonomousAgent) carries the entire response; two guardrail hooks sit around it at different cut points. Two governance mechanisms are wired around the agent:

- A **before-llm-call guardrail** (`TopicPolicyGuardrail`) inspects each incoming user message before the model call. If the message touches a blocked topic (financial-advice, medical-diagnosis, legal-advice by default), the guardrail returns a structured rejection, a `TurnBlocked` event is recorded, and the LLM is never invoked.
- A **before-agent-response guardrail** (`ContentPolicyGuardrail`) validates the agent's candidate reply on every turn: no disallowed phrases, no content that contradicts the agent's declared scope, no responses that begin with a refusal followed by compliance. A malformed or non-compliant reply triggers a retry inside the same task.

The blueprint shows both hooks working independently at different lifecycle moments — input filtering and output validation — around one agent.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a message into the **Message** textarea and clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/turns` and the turn appears in the session timeline.
2. Within ~1 s the turn card transitions to `CHECKING_INPUT` — the topic-policy guardrail result is shown (ALLOWED or BLOCKED). If BLOCKED, the card shows the blocked-topic name and no LLM call was made.
3. If ALLOWED, within ~10–30 s the turn transitions to `GENERATING`. The response appears with a content-policy badge (PASSED, RETRY-1, RETRY-2) indicating how many output iterations the content guardrail required.
4. The user can start a new session or continue sending messages in the same session. The session timeline scrolls and keeps all turn history.
5. Three seeded "demo scenarios" are available from a dropdown — each pre-fills a message sequence that exercises a different control path.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create, send turn, list turns, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle and turn log: opened → active → closed. Source of truth. | `SessionEndpoint`, `SessionWorkflow` | `SessionView` |
| `SessionWorkflow` | `Workflow` | One workflow per session. Steps: `openStep` → `exchangeStep` (repeatable) → `closeStep`. | started by `SessionEndpoint` on session create | `ConversationAgent`, `SessionEntity` |
| `ConversationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives session context in the task definition and the user message as the task instruction; returns `AgentReply`. Guardrails `TopicPolicyGuardrail` (before-llm-call) and `ContentPolicyGuardrail` (before-agent-response) are registered on this agent. | invoked by `SessionWorkflow` | returns reply |
| `TopicPolicyGuardrail` | supporting class | Registered on `ConversationAgent` at the `before-llm-call` hook. Checks the incoming message against the blocked-topic list; rejects if matched. | `ConversationAgent` input | structured rejection / pass-through |
| `ContentPolicyGuardrail` | supporting class | Registered on `ConversationAgent` at the `before-agent-response` hook. Validates the candidate reply for disallowed phrases and scope compliance; rejects if violated. | `ConversationAgent` output | structured rejection / pass-through |
| `SessionView` | `View` | Read model: one row per session plus embedded turn list for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TurnRequest(
    String turnId,
    String userMessage,
    String sessionId,
    Instant sentAt
) {}

record TopicCheckResult(
    TopicCheckOutcome outcome,   // ALLOWED | BLOCKED
    String matchedTopic,         // null when ALLOWED
    String reason
) {}
enum TopicCheckOutcome { ALLOWED, BLOCKED }

record AgentReply(
    String replyText,
    int contentPolicyIterations,  // how many before-agent-response retries were needed
    Instant repliedAt
) {}

record TurnRecord(
    String turnId,
    String userMessage,
    Optional<TopicCheckResult> topicCheck,
    Optional<AgentReply> reply,
    TurnStatus status,
    Instant startedAt,
    Optional<Instant> completedAt
) {}
enum TurnStatus {
    RECEIVED, CHECKING_INPUT, BLOCKED, GENERATING, COMPLETED, FAILED
}

record Session(
    String sessionId,
    String userId,
    List<TurnRecord> turns,
    SessionStatus status,
    Instant openedAt,
    Optional<Instant> closedAt
) {}
enum SessionStatus { OPEN, CLOSED, FAILED }
```

Events on `SessionEntity`: `SessionOpened`, `TurnReceived`, `TopicChecked`, `TurnBlocked`, `GenerationStarted`, `ReplyRecorded`, `TurnFailed`, `SessionClosed`.

Every nullable lifecycle field on `TurnRecord` and `Session` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ userId }` → `{ sessionId }`.
- `POST /api/sessions/{sessionId}/turns` — body `{ userMessage }` → `{ turnId }`.
- `GET /api/sessions/{sessionId}` — full session with turn list.
- `GET /api/sessions/{sessionId}/sse` — Server-Sent Events; one event per state transition.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Agent with Guardrails</title>`.

The App UI tab is a two-column layout: a left rail with the list of sessions (status badge + userId + age + turn count) and a right pane with the selected session's turn timeline — each turn card showing user message, topic-check outcome badge, content-policy iteration count badge (when reply landed), and the agent's reply text.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-llm-call topic guardrail** (`guardrail`, `before-llm-call`): runs on every incoming message before the LLM is called. Checks the message text against a configurable blocked-topic list (financial-advice, medical-diagnosis, legal-advice). On a match, returns a structured `topic-blocked` rejection that causes `SessionWorkflow.exchangeStep` to record `TurnBlocked` on the entity — the model is never invoked, and the block reason is stored for audit.
- **G2 — before-agent-response content guardrail** (`guardrail`, `before-agent-response`): runs on every candidate reply before it leaves the agent loop. Checks for: disallowed phrases (a configurable list stored in `src/main/resources/content-policy/disallowed-phrases.txt`), scope-contradiction (a reply that claims the agent can do something the system prompt says it cannot), and refusal-then-compliance (a reply that opens with a refusal but then provides the requested content anyway). On failure, returns a structured rejection naming the violated check; the agent loop retries within its iteration budget.

## 9. Agent prompts

- `ConversationAgent` → `prompts/conversation-agent.md`. The single decision-making LLM. System prompt declares the agent's scope, its permitted topics, and the output format it must produce. References both guardrails in its behavioral rules so the model understands why it may be retried.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User opens a session, sends a message on an allowed topic; within 30 s the reply appears, the turn status is `COMPLETED`, and `contentPolicyIterations` is 1 (first candidate passed).
2. **J2** — User sends a message containing financial-advice content — the `before-llm-call` guardrail blocks it; the `TurnBlocked` event is recorded; the LLM call log contains no record of the message body; the UI shows a BLOCKED badge.
3. **J3** — Mock LLM mode: the agent's first candidate reply contains a disallowed phrase — the `before-agent-response` guardrail rejects it; the agent retries; the second reply is clean; `contentPolicyIterations` is 2 on the `ReplyRecorded` event.
4. **J4** — Service log confirms a blocked-topic message body never appears in any LLM call log; only the `topic-blocked` event payload appears.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named guardrails-demo demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-guardrails-demo. Java package
io.akka.samples.agentwithguardrailsintegration. Akka 3.6.0. HTTP port 9228.

Components to wire (exactly):

- 1 AutonomousAgent ConversationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/conversation-agent.md>) and
  .capability(TaskAcceptance.of(REPLY_TO_MESSAGE).maxIterationsPerTask(3)).
  Two guardrails registered on this agent:
  * TopicPolicyGuardrail bound to the before-llm-call hook.
  * ContentPolicyGuardrail bound to the before-agent-response hook.
  Output: AgentReply{replyText: String, contentPolicyIterations: int, repliedAt: Instant}.
  The task receives the user message as its instruction text and the session context
  (last 3 turns) as an attachment named "context.txt".

- 1 Workflow SessionWorkflow per sessionId with three step types:
  * openStep — emits SessionOpened on SessionEntity; starts a persistent session. Timeout 5s.
  * exchangeStep — for each turn: emits TurnReceived, then calls
    componentClient.forAutonomousAgent(ConversationAgent.class, "agent-" + sessionId)
    .runSingleTask(TaskDef.instructions(userMessage).attachment("context.txt", contextBytes)).
    On topic block (TopicPolicyGuardrail rejection propagated as a structured error), records
    TurnBlocked on entity instead of calling the agent. On successful reply records
    ReplyRecorded. Timeout 60s per exchange with defaultStepRecovery maxRetries(2)
    .failoverTo(SessionWorkflow::error).
  * closeStep — emits SessionClosed. Timeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  userId: String, turns: List<TurnRecord>, status: SessionStatus, openedAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus enum: OPEN, CLOSED, FAILED.
  Events: SessionOpened{userId}, TurnReceived{turnRequest}, TopicChecked{result},
  TurnBlocked{turnId, topic, reason}, GenerationStarted{turnId},
  ReplyRecorded{turnId, reply}, TurnFailed{turnId, reason}, SessionClosed{}.
  Commands: open, receiveTurn, recordTopicCheck, recordBlock, markGenerating, recordReply,
  failTurn, close, getSession. emptyState() returns Session.initial("") with empty turns
  list and no commandContext() reference (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View SessionView with row type SessionRow (mirrors Session including embedded
  List<TurnRecord> turns). Table updater consumes SessionEntity events. ONE query
  getAllSessions: SELECT * AS sessions FROM session_view. No WHERE status filter —
  Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {userId}; mints sessionId; starts
    SessionWorkflow; returns {sessionId}), POST /sessions/{sessionId}/turns (body
    {userMessage}; mints turnId; triggers exchangeStep; returns {turnId}),
    GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{sessionId} (one session row), GET /sessions/{sessionId}/sse
    (Server-Sent Events forwarded from the view's stream-updates), and three
    /api/metadata/* endpoints serving YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SessionTasks.java declaring one Task<R> constant: REPLY_TO_MESSAGE = Task
  .name("Reply to message")
  .description("Read the user message and session context and produce an AgentReply")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records TurnRequest, TopicCheckResult, TopicCheckOutcome, AgentReply,
  TurnRecord, TurnStatus, Session, SessionStatus.

- TopicPolicyGuardrail.java implementing the before-llm-call hook. Reads the blocked-topic
  list from src/main/resources/content-policy/blocked-topics.txt (one topic keyword per
  line). Checks if the incoming user message contains any blocked keyword
  (case-insensitive). On match returns Guardrail.reject({topic, reason}); on no match
  passes through. The rejection payload names the matched topic.

- ContentPolicyGuardrail.java implementing the before-agent-response hook. Reads the
  disallowed-phrase list from src/main/resources/content-policy/disallowed-phrases.txt.
  Checks the candidate reply for: (1) disallowed phrases, (2) scope contradiction
  (reply claims capabilities the system prompt excludes), (3) refusal-then-compliance
  (reply opens with a refusal string but then provides the content). On any failure
  returns Guardrail.reject({violatedCheck, excerpt}) to force a retry within the
  3-iteration budget.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9228 and
  the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/content-policy/blocked-topics.txt with default entries:
  financial-advice, medical-diagnosis, legal-advice, investment-recommendation,
  prescription-medication.

- src/main/resources/content-policy/disallowed-phrases.txt with 10 example disallowed
  phrases covering common refusal-compliance patterns and brand-dangerous content.

- src/main/resources/sample-events/demo-scenarios.jsonl with 3 seeded demo scenarios:
  (1) allowed-topic happy-path (3 messages on general product questions),
  (2) blocked-topic demo (1 message that matches financial-advice),
  (3) content-policy retry demo (mock-mode only — first reply contains a disallowed
  phrase, second retry passes).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies
  of root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root.

- prompts/conversation-agent.md loaded as the agent system prompt.

- README.md at the project root.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs. App UI tab uses a two-column layout (left = session list; right =
  selected session's turn timeline with per-turn topic-check and content-policy badges).
  Browser title exactly: <title>Akka Sample: Agent with Guardrails</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a
  per-task dispatch on the Task<R> id.
- Per-task mock-response shapes for THIS blueprint:
    reply-to-message.json — 6 AgentReply entries covering normal and edge cases.
      Two entries are deliberately NON-COMPLIANT (contain a disallowed phrase from
      disallowed-phrases.txt) so ContentPolicyGuardrail has work to do. The mock
      selects the non-compliant entry on the first iteration of every 4th session turn
      (modulo seed) so J3 is reproducible.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ConversationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SessionTasks.java MUST
  exist.
- Lesson 4: every workflow step has an explicit stepTimeout (openStep 5s, exchangeStep
  60s, closeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on TurnRecord and Session is Optional<T>.
- Lesson 7: SessionTasks.java with REPLY_TO_MESSAGE = Task.name(...)
  .description(...).resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9228 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS
  overrides (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by
  NodeList index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ConversationAgent).
  Both guardrails are supporting classes — they do not make independent LLM calls.
- The before-llm-call guardrail (TopicPolicyGuardrail) runs BEFORE the model is invoked.
  Verify the generated exchangeStep log shows no LLM call when a topic block fires.
- The before-agent-response guardrail (ContentPolicyGuardrail) is wired via the agent's
  guardrail-configuration block, not as an external check after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
