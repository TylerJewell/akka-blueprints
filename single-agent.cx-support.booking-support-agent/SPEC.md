# SPEC — booking-support-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BookingSupportAgent.
**One-line pitch:** A customer sends a natural-language message about their travel booking; one AI agent looks up the booking, answers questions, and carries out permitted changes (seat selection, date modification, cancellation) — returning a typed reply with the action taken and a plain-language explanation.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `BookingSupportAgent` (AutonomousAgent) handles the entire customer interaction turn; surrounding components sanitize input and audit output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw customer message and the agent call — so the model never sees payment-card numbers, government identifiers, or other identifiers that appear in customer messages.
- A **before-tool-call guardrail** intercepts every destructive tool call (modify booking, cancel booking, issue refund credit). It verifies the target booking belongs to the requesting customer id, that the requested change is within the booking's fare-rule permissions, and that the session has not already applied a same-type change in this turn. A blocked tool call returns a structured rejection to the agent loop so the agent can respond accordingly.
- A **CI gate** backed by LLM-judge test assertions runs on every pull request. `JudgeAssertions.java` defines a set of session scenarios; each scenario runs the agent in mock mode and evaluates the reply with a judge prompt (does the reply answer the question? does it hallucinate booking details? is the language appropriate?).

The blueprint shows that one agent can make consequential booking changes while governance prevents cross-customer access and unintended mutations.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects a **customer id** from a dropdown of seeded customers (or types one) and types a message in the **Message** textarea, or picks one of four seeded scenarios (look up flight, change seat, cancel booking, ask about refund policy).
2. The user clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/turns` (creating the session if needed) and receives a `turnId`.
3. The turn card appears in the live session panel in `RECEIVED` state. Within ~1 s it transitions to `SANITIZED` — the redacted message is visible with PII-category chips.
4. Within ~10–30 s the agent turn completes. The card transitions to `AGENT_REPLIED`. The reply appears: the agent's natural-language response, a list of any tool calls made (with outcome: ALLOWED / BLOCKED), and the affected booking's new state if anything changed.
5. The user can send another message in the same session; the session panel keeps prior turns visible as scrollable history.
6. Any blocked tool call shows a red BLOCKED chip with the guardrail reason; the agent's reply to the customer is shown regardless.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SupportEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create session, submit turn, list sessions, get session, SSE; serves `/api/metadata/*`. | — | `BookingSessionEntity`, `BookingSessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BookingSessionEntity` | `EventSourcedEntity` | Per-session lifecycle: created → received → sanitized → agent_replied. Source of truth for conversation history, tool-call log, and booking mutation log. | `SupportEndpoint`, `PiiSanitizer`, `BookingSessionWorkflow` | `BookingSessionView` |
| `PiiSanitizer` | `Consumer` | Subscribes to `CustomerMessageReceived` events; redacts PII from message text; calls `BookingSessionEntity.attachSanitized`. | `BookingSessionEntity` events | `BookingSessionEntity` |
| `BookingSessionWorkflow` | `Workflow` | One workflow per session turn. Steps: `awaitSanitizedStep` → `agentTurnStep` → `recordTurnStep`. | started by `PiiSanitizer` after sanitized event lands | `BookingSupportAgent`, `BookingSessionEntity` |
| `BookingSupportAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the customer message (sanitized), session history, and booking context as the task; returns `AgentReply`. Registered before-tool-call guardrail validates destructive calls. | invoked by `BookingSessionWorkflow` | returns `AgentReply` |
| `ToolCallGuardrail` | supporting class | `before-tool-call` hook wired on `BookingSupportAgent`. Checks ownership, fare-rule permission, and duplicate-change guard. Returns ALLOW or structured BLOCK. | — | agent loop |
| `BookingSessionView` | `View` | Read model: one row per session turn for the UI. | `BookingSessionEntity` events | `SupportEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CustomerMessage(
    String sessionId,
    String turnId,
    String customerId,
    String rawMessage,
    Instant receivedAt
) {}

record SanitizedMessage(
    String sanitizedText,
    List<String> piiCategoriesFound
) {}

record BookingRecord(
    String bookingRef,
    String customerId,
    String flightNumber,
    String origin,
    String destination,
    Instant departureAt,
    Instant arrivalAt,
    String seatNumber,
    BookingStatus bookingStatus,
    String fareClass,
    boolean cancellationPermitted,
    boolean seatChangePermitted
) {}
enum BookingStatus { CONFIRMED, MODIFIED, CANCELLED, REFUND_PENDING }

record ToolCall(
    String toolName,
    String targetBookingRef,
    String requestedChange,
    ToolCallOutcome outcome,
    String guardrailReason    // null when ALLOWED
) {}
enum ToolCallOutcome { ALLOWED, BLOCKED }

record AgentReply(
    String replyText,
    List<ToolCall> toolCalls,
    Optional<BookingRecord> updatedBooking,
    Instant repliedAt
) {}

record SessionTurn(
    String turnId,
    Optional<CustomerMessage> message,
    Optional<SanitizedMessage> sanitized,
    Optional<AgentReply> reply,
    TurnStatus status,
    Instant createdAt,
    Optional<Instant> completedAt
) {}
enum TurnStatus { RECEIVED, SANITIZED, AGENT_REPLIED, FAILED }

record BookingSession(
    String sessionId,
    String customerId,
    List<SessionTurn> turns,
    SessionStatus sessionStatus,
    Instant createdAt,
    Optional<Instant> closedAt
) {}
enum SessionStatus { OPEN, CLOSED, FAILED }
```

Events on `BookingSessionEntity`: `SessionCreated`, `CustomerMessageReceived`, `MessageSanitized`, `AgentTurnCompleted`, `SessionClosed`, `TurnFailed`.

Every nullable lifecycle field on `SessionTurn` and `BookingSession` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ customerId }` → `{ sessionId }`.
- `POST /api/sessions/{sessionId}/turns` — body `{ customerId, rawMessage }` → `{ turnId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{sessionId}` — one session with full turn history.
- `GET /api/sessions/{sessionId}/sse` — Server-Sent Events; one event per turn state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Customer Support Booking Agent</title>`.

The App UI tab is a two-column layout: a left rail with the active session panel (message input, send button, and scrollable turn history) and a right pane showing the selected turn's detail — sanitized message with PII chips, tool-call log with ALLOWED/BLOCKED outcomes and guardrail reasons, agent reply text, and updated booking card if applicable.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`, applied inside `ToolCallGuardrail`): intercepts every call to `lookUpBooking`, `modifyBooking`, `cancelBooking`, and `issueRefundCredit` before execution. Verifies (1) the target `bookingRef` belongs to the `customerId` in the current session, (2) the requested change type is permitted by the booking's fare-rule flags (`cancellationPermitted`, `seatChangePermitted`), and (3) the session has not already applied an identical change in this turn (duplicate-mutation guard). A failing check returns a structured BLOCK with a reason; the agent loop receives the block and replies to the customer accordingly. The tool call is never executed for blocked operations.
- **S1 — PII sanitizer** (`sanitizer`, `pii`, applied inside `PiiSanitizer` Consumer): redacts payment-card numbers, government identifiers, phone numbers, emails, and account-number-like tokens from the customer's raw message before any LLM call. The raw message is retained on the entity for audit but is never the input the model receives.
- **H1 — CI gate** (`ci-gate`, `test-gate`, enforced at build time by `JudgeAssertions.java`): a set of annotated test scenarios, each pairing an input session with a judge prompt. On each CI run the agent processes each scenario in mock mode; the judge prompt evaluates whether the reply is grounded in the booking data, avoids hallucinated flight details, and maintains appropriate tone. A judge-fail blocks the build.

## 9. Agent prompts

- `BookingSupportAgent` → `prompts/booking-support-agent.md`. The single decision-making LLM. System prompt instructs it to answer customer questions about their bookings using only the data returned by the booking-lookup tool, carry out permitted changes via the appropriate tool, and decline gracefully when the guardrail blocks a tool call.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer asks "What time does my flight depart?" → agent calls `lookUpBooking`, guardrail passes, reply includes departure time from the booking record.
2. **J2** — Customer asks to cancel a booking they own and the booking's `cancellationPermitted = true` → guardrail passes → `cancelBooking` executes → reply confirms; booking status changes to `CANCELLED`.
3. **J3** — Customer asks to cancel a booking whose `customerId` does not match the session's `customerId` → guardrail blocks with reason `OWNERSHIP_VIOLATION` → `cancelBooking` is never executed → agent tells the customer it cannot process the request.
4. **J4** — Customer asks to change their seat on a booking whose `seatChangePermitted = false` (non-flexible fare) → guardrail blocks with reason `FARE_RULE_VIOLATION` → agent explains that the fare does not allow seat changes.
5. **J5** — Customer message contains `4111 1111 1111 1111` (test card number) → LLM call log shows only `[REDACTED-PAYMENT-CARD]`; entity's `rawMessage` retains the original.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named booking-support-agent demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-booking-support-agent. Java package
io.akka.samples.customersupportbookingagent. Akka 3.6.0. HTTP port 9578.

Components to wire (exactly):

- 1 AutonomousAgent BookingSupportAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/booking-support-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_SUPPORT_TURN).maxIterationsPerTask(4)). The task
  carries the sanitized customer message, session history summary, and available booking
  context in the task instructions. The agent is configured with a before-tool-call guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  Output: AgentReply{replyText: String, toolCalls: List<ToolCall>,
  updatedBooking: Optional<BookingRecord>, repliedAt: Instant}.

- 1 Workflow BookingSessionWorkflow per (sessionId, turnId) with three steps:
  * awaitSanitizedStep — polls BookingSessionEntity.getSession every 1s; on the current
    turn's sanitized field being present advances to agentTurnStep.
    WorkflowSettings.stepTimeout 15s.
  * agentTurnStep — emits turn status update, then calls componentClient.forAutonomousAgent(
    BookingSupportAgent.class, "support-" + sessionId).runSingleTask(
      TaskDef.instructions(formatTurnContext(session, sanitizedMessage))
    ) — returns a taskId, then forTask(taskId).result(HANDLE_SUPPORT_TURN) to fetch the
    reply. On success calls BookingSessionEntity.recordAgentTurn(reply).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(BookingSessionWorkflow::error).
  * recordTurnStep — updates the turn status to AGENT_REPLIED and closes the workflow.
    WorkflowSettings.stepTimeout 5s. error step transitions the entity turn to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity BookingSessionEntity (one per sessionId). State BookingSession
  {sessionId: String, customerId: String, turns: List<SessionTurn>, sessionStatus:
  SessionStatus, createdAt: Instant, closedAt: Optional<Instant>}. SessionStatus enum: OPEN,
  CLOSED, FAILED. TurnStatus enum: RECEIVED, SANITIZED, AGENT_REPLIED, FAILED. Events:
  SessionCreated{customerId}, CustomerMessageReceived{message: CustomerMessage},
  MessageSanitized{turnId, sanitized: SanitizedMessage}, AgentTurnCompleted{turnId,
  reply: AgentReply}, SessionClosed{}, TurnFailed{turnId, reason: String}. Commands:
  createSession, receiveMessage, attachSanitized, recordAgentTurn, closeSession, failTurn,
  getSession. emptyState() returns BookingSession.initial("") with no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer PiiSanitizer subscribed to BookingSessionEntity events; on
  CustomerMessageReceived runs a regex+heuristic pipeline (payment-card numbers,
  government-id-like tokens, phone numbers, emails, account-number-like tokens) over
  rawMessage, computes piiCategoriesFound, builds SanitizedMessage, then calls
  BookingSessionEntity.attachSanitized(turnId, sanitized). After attachSanitized lands,
  the same Consumer starts a BookingSessionWorkflow with id = "turn-" + turnId.

- 1 View BookingSessionView with row type SessionRow (mirrors BookingSession). Table updater
  consumes BookingSessionEntity events. ONE query getAllSessions: SELECT * AS sessions FROM
  session_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * SupportEndpoint at /api with POST /sessions (body {customerId}; mints sessionId; calls
    BookingSessionEntity.createSession; returns {sessionId}), POST /sessions/{sessionId}/turns
    (body {customerId, rawMessage}; mints turnId; calls BookingSessionEntity.receiveMessage;
    returns {turnId}), GET /sessions (list from getAllSessions, sorted newest-first), GET
    /sessions/{sessionId} (one session), GET /sessions/{sessionId}/sse (Server-Sent Events
    forwarded from the view's stream-updates), and three /api/metadata/* endpoints serving
    the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- BookingSupportTasks.java declaring one Task<R> constant: HANDLE_SUPPORT_TURN =
  Task.name("Handle support turn").description("Answer the customer's question or carry out
  the requested booking change, using only the booking tools provided")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records CustomerMessage, SanitizedMessage, BookingRecord, ToolCall, ToolCallOutcome,
  AgentReply, SessionTurn, TurnStatus, BookingSession, SessionStatus, BookingStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. On each proposed tool call
  it (1) checks the target bookingRef's customerId matches the session customerId
  (OWNERSHIP_VIOLATION on failure), (2) checks fare-rule flags for modify/cancel calls
  (FARE_RULE_VIOLATION on failure), and (3) checks no identical change has already been
  applied in this turn (DUPLICATE_MUTATION on failure). Either passes the call through or
  returns Guardrail.block(<structured-reason>).

- BookingStore.java — in-process read-only store backed by bookings.jsonl. Loaded at startup.
  Provides lookUpBooking(bookingRef): Optional<BookingRecord> and
  listCustomerBookings(customerId): List<BookingRecord>. The agent's tool implementations
  read from this store; mutations are recorded on the entity but do NOT write back to the
  store in this baseline (the store is seeded read-only data).

- JudgeAssertions.java — test class that defines 5 session scenarios. Each scenario: a
  fixed customerId + message + expected reply property (e.g., "reply must contain departure
  time", "reply must not mention another customer's booking"). Runs the agent in mock-LLM
  mode; evaluates the reply with a judge prompt via the configured model provider. Fails the
  test if the judge scores the reply below the threshold. This is the CI gate (H1).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9578 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The BookingSupportAgent.definition()
  binds the configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/bookings.jsonl with 8 seeded BookingRecord entries across
  3 customers (C001, C002, C003): a mix of confirmed flexible fares, confirmed non-flexible
  fares, and one already-cancelled booking. Each record has realistic flight numbers,
  origins, and destinations. C001 has 3 bookings; C002 has 3 bookings; C003 has 2 bookings.
  Includes 2 bookings with cancellationPermitted=false to exercise the fare-rule guardrail
  path.

- src/main/resources/sample-events/seed-turns.jsonl with 4 paired (customerId, message)
  examples: "What time does my BK-001 flight depart?", "Can I change my seat on BK-002?",
  "Please cancel booking BK-003", "What is your refund policy?" — one for each seeded scenario.
  Two of them contain a PII-like token in the message body so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, S1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = full-execution
  (the agent actually mutates bookings), oversight.human_in_loop = false
  (automated changes; humans review audit log), failure.failure_modes including
  "cross-customer-booking-access", "hallucinated-booking-detail", "pii-leakage-via-llm",
  "double-cancel"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/booking-support-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Customer Support Booking Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  session panel with message input and scrollable turn history; right = selected-turn detail
  with sanitized message, PII chips, tool-call log, agent reply, and updated booking card).
  Browser title exactly: <title>Akka Sample: Customer Support Booking Agent</title>.
  No subtitle on the Overview tab.

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
    handle-support-turn.json — 8 AgentReply entries covering look-up, seat-change,
      cancellation, refund-policy-inquiry, and error scenarios. Each entry has a replyText
      paragraph and a toolCalls list. 2 entries include a ToolCall with outcome=BLOCKED
      (one OWNERSHIP_VIOLATION, one FARE_RULE_VIOLATION) to exercise the guardrail path.
      1 entry has a hallucinated flight number not present in bookings.jsonl — the CI judge
      assertion catches it. MockModelProvider.seedFor(sessionId) makes selection
      deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. BookingSupportAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BookingSupportTasks.java
  MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (agentTurnStep 60s,
  awaitSanitizedStep 15s, recordTurnStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on SessionTurn and BookingSession is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...)
  or .isPresent().
- Lesson 7: BookingSupportTasks.java with HANDLE_SUPPORT_TURN = Task.name(...)
  .description(...).resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9578 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (BookingSupportAgent).
  The CI gate (JudgeAssertions.java) uses the same agent in mock mode — it does not
  introduce a second live LLM call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external wrapper. It intercepts tool calls before execution, not after.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
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
