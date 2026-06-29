# SPEC — realtime-conversational-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RealtimeConversationalAgent.
**One-line pitch:** A customer opens a session; one AI agent handles every turn of the conversation — greeting, question answering, and graceful close — while a `before-agent-response` guardrail checks every candidate reply against the operator's content policy before the customer sees it.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the customer-experience support domain. One `ConversationalAgent` (AutonomousAgent) handles the entire dialogue; surrounding components manage session lifecycle, enforce the content policy, and compute a post-session quality score.

One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates every candidate reply on every turn: the reply must be within the configured topic scope, must not contain prohibited content categories (profanity, competitor brand names, medical/legal/financial advice beyond defined limits, personally identifiable information the system itself injected into the prompt), and must not exceed the maximum token budget per turn. A rejected reply triggers an in-loop retry within the same task iteration.

The blueprint shows that a real-time conversational system can be safe without a second model call for moderation — one deterministic hook, running synchronously before the reply leaves the agent, is sufficient for customer-facing voice and text.

## 3. User-facing flows

The user opens the App UI tab.

1. The user enters a **Customer ID** (or uses the seeded demo ID) and clicks **Start session**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
2. The session card appears in the live list in `GREETING` state. Within ~1–5 s, the agent emits the opening greeting. The card transitions to `ACTIVE` and the right pane shows the greeting as the first turn.
3. The user types a message in the **Customer message** textarea and clicks **Send**. The UI POSTs to `/api/sessions/{id}/turns`. The turn is appended to the turn history immediately in `WAITING_FOR_AGENT` state.
4. Within ~5–30 s, the agent replies. The turn transitions to `AGENT_REPLIED` and the reply text appears in the conversation thread.
5. The user continues sending messages. Each round-trip is one turn; the turn history grows in the right pane.
6. The user clicks **End session**. The UI POSTs to `/api/sessions/{id}/end`. The session transitions to `SUMMARIZING`; within ~1 s the `TurnSummarizer` finishes and the session moves to `CLOSED`. A session-quality score (1–5) and a one-paragraph summary appear on the card.
7. Closed sessions remain in the list so operators can review the full turn history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — start, send turn, end, list, get, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: greeting → active → summarizing → closed. Holds the full turn history. Source of truth. | `SessionEndpoint`, `SessionWorkflow` | `SessionView` |
| `SessionWorkflow` | `Workflow` | One workflow per session. Steps: `greetStep` → `activeStep` (loop) → `summarizeStep`. | started by `SessionEndpoint` on POST /sessions | `ConversationalAgent`, `SessionEntity`, `TurnSummarizer` |
| `ConversationalAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the conversation history and the customer's latest message as a task attachment; returns `AgentTurn`. | invoked by `SessionWorkflow` per turn | returns `AgentTurn` |
| `ResponseGuardrail` | guardrail hook | Registered on `ConversationalAgent` via before-agent-response. Checks topic scope, prohibited content, and token budget. | invoked by agent loop on each candidate reply | pass-through or rejection |
| `TurnSummarizer` | supporting class | Deterministic post-session scorer (no LLM call). Accepts the completed turn list; returns `SessionSummary`. | invoked by `SessionWorkflow.summarizeStep` | `SessionEntity` |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CustomerMessage(
    String turnId,
    String text,
    Instant receivedAt
) {}

record AgentTurn(
    String turnId,
    String replyText,
    boolean guardrailTriggered,   // true if at least one iteration was rejected
    int iterationsUsed,
    Instant repliedAt
) {}

record ConversationTurn(
    String turnId,
    CustomerMessage customer,
    Optional<AgentTurn> agent,
    TurnStatus status,
    Instant startedAt
) {}
enum TurnStatus { WAITING_FOR_AGENT, AGENT_REPLIED, FAILED }

record SessionSummary(
    int qualityScore,        // 1..5
    String summaryText,      // 1–3 sentences
    int totalTurns,
    int guardrailEvents,
    Instant summarizedAt
) {}

record Session(
    String sessionId,
    String customerId,
    String agentPersona,
    List<ConversationTurn> turns,
    Optional<SessionSummary> summary,
    SessionStatus status,
    Instant openedAt,
    Optional<Instant> closedAt
) {}

enum SessionStatus {
    GREETING, ACTIVE, WAITING_FOR_AGENT, SUMMARIZING, CLOSED, FAILED
}
```

Events on `SessionEntity`: `SessionOpened`, `GreetingEmitted`, `CustomerTurnReceived`, `AgentReplied`, `TurnFailed`, `SessionEndRequested`, `SessionSummarized`, `SessionFailed`.

Every nullable lifecycle field on the `Session` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ customerId, agentPersona? }` → `{ sessionId }`.
- `POST /api/sessions/{id}/turns` — body `{ text }` → `{ turnId }`. Queues the customer message; agent reply arrives via SSE.
- `POST /api/sessions/{id}/end` — body `{}` → `204`. Triggers summarize.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session with full turn history.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition and per completed turn.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Realtime Conversational Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of sessions (status pill + session ID + customer ID + turn count + age) and a right pane with the selected session's conversation thread — each turn showing the customer text and the agent reply, guardrail-triggered badge if applicable, and the session summary when closed.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `ConversationalAgent`. Asserts the candidate reply is within the configured topic scope (using a keyword-and-category classifier), does not contain prohibited categories (profanity, competitor brand names, unsolicited medical/legal/financial guidance, injected PII patterns from the prompt context), and does not exceed the `MAX_REPLY_TOKENS` limit. On failure, returns a structured `policy-violation` error naming the category to the agent loop so the task retries within its 3-iteration budget. Passing replies flow through to the workflow.

## 9. Agent prompts

- `ConversationalAgent` → `prompts/conversational-agent.md`. The single decision-making LLM. System prompt instructs it to hold a helpful, terse conversation with the customer, stay within the operator-defined topic scope, and always produce a reply that a `before-agent-response` guardrail will accept.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer opens a session; the greeting arrives; three customer messages produce three agent replies; the session closes with a quality score visible on the card.
2. **J2** — On one turn the agent's first candidate reply triggers the guardrail (mock LLM path); the second iteration produces a passing reply; the customer sees only the passing reply; the `SessionEntity` log never contains the rejected text.
3. **J3** — A session in which the agent triggered the guardrail on every turn (all iterations rejected on one turn, causing the turn to fail) receives a quality score of 1; the UI flags the card with a red border.
4. **J4** — A browser tab joins mid-session; the SSE stream replays the prior turns from the view; the new browser sees the full history without a page reload.
5. **J5** — A session that ends normally has all `ConversationTurn` entries in `AGENT_REPLIED` status and `Session.status = CLOSED`; the `SessionSummary` is present with `totalTurns` equal to the number of turns in the history.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named realtime-conversational-agent demonstrating the single-agent
× cx-support cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact single-agent-cx-support-realtime-voice. Java package
io.akka.samples.realtimeconversationalagent. Akka 3.6.0. HTTP port 9397.

Components to wire (exactly):

- 1 AutonomousAgent ConversationalAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/conversational-agent.md>) and
  .capability(TaskAcceptance.of(ConversationTasks.REPLY_TO_CUSTOMER).maxIterationsPerTask(3)).
  On each task the agent receives the conversation history (prior turns, JSON-serialised) as
  its instruction context and the customer's latest message as a task ATTACHMENT named
  "customer-message.txt" (TaskDef.attachment — NOT inline prompt text). Output: AgentTurn{
  turnId: String, replyText: String, guardrailTriggered: boolean, iterationsUsed: int,
  repliedAt: Instant}. The agent is configured with a before-agent-response guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries the response within its 3-iteration budget.

- 1 Workflow SessionWorkflow per sessionId with steps:
  * greetStep — calls ConversationalAgent with a GREET_CUSTOMER task (an empty attachment
    named "customer-message.txt" with bytes "<<GREETING>>"); on success calls
    SessionEntity.recordGreeting(agentTurn). WorkflowSettings.stepTimeout 15s.
  * activeStep — a pause step: the workflow suspends and waits for an external signal
    (customer turn). When SessionEndpoint calls SessionWorkflow.submitTurn(customerMessage),
    the workflow advances to converseTurnStep. When SessionEndpoint calls
    SessionWorkflow.requestEnd(), it advances to summarizeStep.
  * converseTurnStep — emits CustomerTurnReceived, then calls
    componentClient.forAutonomousAgent(ConversationalAgent.class, "agent-" + sessionId)
    .runSingleTask(TaskDef.instructions(formatHistory(session.turns()))
      .attachment("customer-message.txt", turn.text().getBytes()))
    to fetch the agent reply. On success calls SessionEntity.recordAgentReply(turnId, agentTurn).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(SessionWorkflow::error).
  * summarizeStep — calls TurnSummarizer.summarize(session.turns()); emits SessionSummarized.
    WorkflowSettings.stepTimeout 5s.
  * error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  customerId: String, agentPersona: String, turns: List<ConversationTurn>,
  summary: Optional<SessionSummary>, status: SessionStatus, openedAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus enum: GREETING, ACTIVE, WAITING_FOR_AGENT,
  SUMMARIZING, CLOSED, FAILED. Events: SessionOpened{customerId, agentPersona, openedAt},
  GreetingEmitted{agentTurn}, CustomerTurnReceived{customerMessage},
  AgentReplied{turnId, agentTurn}, TurnFailed{turnId, reason}, SessionEndRequested{},
  SessionSummarized{summary}, SessionFailed{reason}. Commands: open, recordGreeting,
  receiveCustomerTurn, recordAgentReply, failTurn, requestEnd, recordSummary, fail,
  getSession. emptyState() returns Session.initial("") with turns = List.of(),
  summary = Optional.empty() and no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 View SessionView with row type SessionRow (mirrors Session minus the raw text of
  each customer message — the entity holds those; the view holds summarized turn metadata
  for the list). Table updater consumes SessionEntity events. ONE query getAllSessions:
  SELECT * AS sessions FROM session_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {customerId, agentPersona?}; mints
    sessionId; calls SessionEntity.open; starts SessionWorkflow; returns {sessionId}),
    POST /sessions/{id}/turns (body {text}; mints turnId; calls
    SessionWorkflow.submitTurn(customerMessage); returns {turnId}),
    POST /sessions/{id}/end (body {}; calls SessionWorkflow.requestEnd(); returns 204),
    GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{id} (one row with full turn history),
    GET /sessions/sse (Server-Sent Events forwarded from the view's stream-updates), and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ConversationTasks.java declaring two Task<R> constants:
    REPLY_TO_CUSTOMER = Task.name("Reply to customer")
      .description("Read the conversation history and the customer message; produce an AgentTurn reply")
      .resultConformsTo(AgentTurn.class);
    GREET_CUSTOMER = Task.name("Greet customer")
      .description("Produce an opening greeting AgentTurn for this session")
      .resultConformsTo(AgentTurn.class);
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records CustomerMessage, AgentTurn, ConversationTurn, TurnStatus, SessionSummary,
  Session, SessionStatus.

- ResponseGuardrail.java implementing the before-agent-response hook. Reads the candidate
  AgentTurn from the LLM response, runs the three checks listed in eval-matrix.yaml G1
  (topic scope, prohibited content categories, token budget), and either passes the response
  through or returns Guardrail.reject(<structured-error>) naming the violated category to
  force the agent loop to retry.

- TurnSummarizer.java — pure deterministic logic (no LLM). Inputs: List<ConversationTurn>.
  Outputs: SessionSummary. Scoring rubric: +1 for each turn where guardrailTriggered=false,
  capped 1–5 relative to total turn count; penalty of -1 per failed turn; summaryText is
  a template sentence with the counts. Javadoc on the class documents the rubric.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9397 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ConversationalAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/faq-entries.jsonl with 3 seeded FAQ knowledge base sets:
  a 6-topic general retail support FAQ, a 5-topic account management FAQ, and a 4-topic
  returns-and-refunds FAQ.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 paired conversation
  seeds: a 3-turn retail support exchange, a 4-turn account inquiry exchange, and a 2-turn
  returns exchange. Each contains plausible customer messages and expected agent reply patterns.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii pre-filled,
  decisions.authority_level = fully-autonomous (the agent's replies are the customer's
  actual experience — no human reads them before delivery), oversight.human_on_loop = true
  (an operator can review session logs), failure.failure_modes including
  "policy-violating-reply", "off-topic-hallucination", "guardrail-exhaustion",
  "session-summary-mismatch"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/conversational-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Realtime Conversational Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session detail with conversation thread,
  each turn showing customer text and agent reply, guardrail-triggered badge if applicable,
  and session summary when closed). Browser title exactly:
  <title>Akka Sample: Realtime Conversational Agent</title>. No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId, turnIndex)),
  and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    reply-to-customer.json — 10 AgentTurn entries covering typical retail, account, and
      returns replies. Each entry has a non-empty replyText paragraph, guardrailTriggered
      set to false for 8 entries, and iterationsUsed = 1. Plus 2 deliberately POLICY-VIOLATING
      entries (one with profanity in replyText; one that references a competitor brand by
      name) — the guardrail blocks both, exercising the retry path. The mock selects a
      violating entry on the FIRST iteration of every 4th turn (modulo seed) so J2 is
      reproducible.
    greet-customer.json — 4 greeting AgentTurn entries, all well-formed, iterationsUsed = 1.
- A MockModelProvider.seedFor(sessionId, turnIndex) helper makes per-session, per-turn
  selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ConversationalAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ConversationTasks.java MUST
  exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (greetStep
  15s, converseTurnStep 60s, summarizeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Session row record is Optional<T>. The
  view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ConversationTasks.java with REPLY_TO_CUSTOMER and GREET_CUSTOMER Task<R>
  constants is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9397 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ConversationalAgent).
  The session summarizer (TurnSummarizer.java) is rule-based and does NOT make an LLM call
  — keeping the pattern's "one agent" promise honest.
- The customer message is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated converseTurnStep uses TaskDef.attachment(...) and not
  string interpolation into the instruction text.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
