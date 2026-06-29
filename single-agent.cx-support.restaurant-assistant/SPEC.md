# SPEC — restaurant-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Restaurant Assistant.
**One-line pitch:** A customer messages a restaurant's digital assistant; one AI agent answers menu questions, accepts reservations, and places orders — with two guardrails ensuring every customer-facing reply is accurate and every write is valid before it commits.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `RestaurantAssistantAgent` (AutonomousAgent) handles the full customer conversation; the surrounding components manage session state and protect both the customer and the restaurant's data integrity. Two governance mechanisms are wired around the agent:

- A **before-agent-response guardrail** validates each reply before it reaches the customer: menu items cited by the agent must exist in the current catalog, prices quoted must be accurate, and the response must not contain unsolicited upsell language outside an approved template. A failing response causes the agent to retry within its iteration budget.
- A **before-tool-call guardrail** validates every reservation or order write before it is committed: reservation dates must be in the future, party sizes must be within the restaurant's configured range, and order line items must reference valid catalog item ids. An invalid tool call is rejected with a structured error so the agent can correct the request rather than silently dropping it.

The blueprint shows that customer-service agents require guardrails at two distinct cut points: one protecting the customer (what they hear) and one protecting the business (what gets written).

## 3. User-facing flows

The user opens the App UI tab.

1. The customer types a message into the **Chat** input on the App UI tab. The UI POSTs to `/api/sessions/{sessionId}/messages` (creating a new session on first message).
2. The response appears in the chat thread. The agent may answer a question ("Our mushroom risotto contains no meat and is marked gluten-friendly"), suggest a reservation, or confirm an order.
3. To make a reservation, the customer says something like "Book a table for 4 on Friday at 7 pm". The agent collects the required fields and calls the `makeReservation` tool. The `before-tool-call` guardrail checks the date and party size before the write lands. The session card shows `RESERVATION_HELD`.
4. To place an order (takeaway or dine-in with a session open), the customer says "I'd like the risotto and two glasses of the house white". The agent builds an order from catalog items and calls the `submitOrder` tool. The `before-tool-call` guardrail validates each line item. The session card shows `ORDER_PLACED`.
5. The live session list on the left reflects status transitions in real time via SSE.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create, message, list, get, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: open → active → reservation-held / order-placed → closed. Source of truth for chat history, cart, and reservation. | `SessionEndpoint`, `SessionWorkflow` | `SessionView` |
| `SessionWorkflow` | `Workflow` | One workflow per session. Steps: `agentStep` → `commitStep` (when a write-tool fires) → `closeStep`. | started by `SessionEndpoint` on first message | `RestaurantAssistantAgent`, `SessionEntity` |
| `RestaurantAssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the current session context and customer message; returns `AssistantResponse` carrying a reply and optional tool call. | invoked by `SessionWorkflow` | returns response |
| `ResponseGuardrail` | guardrail class | before-agent-response hook: validates reply accuracy against the menu catalog. | `RestaurantAssistantAgent` | — |
| `ToolCallGuardrail` | guardrail class | before-tool-call hook: validates reservation/order writes before they are committed. | `RestaurantAssistantAgent` | — |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MenuItem(
    String itemId,
    String name,
    String category,       // STARTER, MAIN, DESSERT, DRINK
    String description,
    double priceGBP,
    boolean vegetarian,
    boolean vegan,
    boolean glutenFriendly,
    List<String> allergens
) {}

record CustomerMessage(
    String messageId,
    String role,           // CUSTOMER, ASSISTANT
    String text,
    Instant sentAt
) {}

record ReservationRequest(
    String date,           // ISO-8601 date e.g. "2026-07-04"
    String time,           // HH:mm e.g. "19:00"
    int partySize,
    String guestName,
    String contactPhone
) {}

record Reservation(
    String reservationId,
    ReservationRequest details,
    Instant confirmedAt
) {}

record OrderLineItem(
    String itemId,
    String itemName,
    int quantity,
    double unitPriceGBP
) {}

record Order(
    String orderId,
    List<OrderLineItem> lines,
    OrderType type,        // DINE_IN, TAKEAWAY
    double totalGBP,
    Instant placedAt
) {}
enum OrderType { DINE_IN, TAKEAWAY }

record AssistantResponse(
    String replyText,
    Optional<String> toolName,       // "makeReservation" | "submitOrder" | null
    Optional<ReservationRequest> reservationArg,
    Optional<List<OrderLineItem>> orderArg,
    Instant respondedAt
) {}

record Session(
    String sessionId,
    List<CustomerMessage> messages,
    Optional<Reservation> reservation,
    Optional<Order> order,
    SessionStatus status,
    Instant openedAt,
    Optional<Instant> closedAt
) {}

enum SessionStatus {
    OPEN, ACTIVE, RESERVATION_HELD, ORDER_PLACED, CLOSED, FAILED
}
```

Events on `SessionEntity`: `SessionOpened`, `MessageAdded`, `AgentReplied`, `ReservationConfirmed`, `OrderCommitted`, `SessionClosed`, `SessionFailed`.

Every nullable lifecycle field on `Session` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ guestName?, contactPhone? }` → `{ sessionId }`. Creates a new session.
- `POST /api/sessions/{id}/messages` — body `{ text }` → `{ messageId, sessionId }`. Sends a customer message; triggers the agent turn.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session with full message history.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Restaurant Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of sessions (status pill + most-recent message excerpt + age) and a right pane with the selected session's chat thread plus a bottom-anchored input bar. The chat thread shows alternating CUSTOMER and ASSISTANT bubbles; when a reservation or order commits, a confirmation card appears inline in the thread.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`ResponseGuardrail`): runs on every turn of `RestaurantAssistantAgent`. Checks that (1) any menu item name the agent cites exists in the in-process catalog by exact match, (2) any price quoted matches the catalog price within £0.01, and (3) the response does not contain prohibited phrases (competitor names, free-item offers not in an approved list). On failure returns a structured rejection naming the failed check; the agent loop retries within its 3-iteration budget.
- **G2 — before-tool-call guardrail** (`ToolCallGuardrail`): runs before any `makeReservation` or `submitOrder` tool call is forwarded to `SessionWorkflow`. For `makeReservation`: asserts date is in the future (at least 1 hour from now), party size is between 1 and 20 inclusive, and `time` is a valid HH:mm in the restaurant's operating hours. For `submitOrder`: asserts every `itemId` exists in the catalog and quantity is ≥ 1. Rejects with a structured error on failure so the agent can correct the tool arguments rather than letting an invalid write land.

## 9. Agent prompts

- `RestaurantAssistantAgent` → `prompts/restaurant-assistant.md`. The single decision-making LLM. System prompt instructs it to answer from the menu catalog context, collect reservation or order details conversationally, and call the correct tool when a customer confirms.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer asks "What vegetarian mains do you have?"; agent returns the correct subset of the seeded menu within one turn.
2. **J2** — Customer requests a table for 4; agent collects date, time, name, and phone; calls `makeReservation`; `ToolCallGuardrail` approves; session transitions to `RESERVATION_HELD`.
3. **J3** — Customer attempts a reservation for 30 people (over the 20-person limit); `ToolCallGuardrail` rejects the tool call; agent replies apologetically and asks for a corrected party size; no write lands.
4. **J4** — Agent (on mock path) cites a menu item that does not exist in the catalog; `ResponseGuardrail` rejects the response; agent retries and returns an accurate reply; the invalid response never reaches the customer.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named restaurant-assistant demonstrating the single-agent × cx-support cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-restaurant-assistant. Java package
io.akka.samples.restaurantassistant. Akka 3.6.0. HTTP port 9309.

Components to wire (exactly):

- 1 AutonomousAgent RestaurantAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/restaurant-assistant.md>) and
  .capability(TaskAcceptance.of(HANDLE_MESSAGE).maxIterationsPerTask(3)). The task receives
  the customer message and session context (current reservation state, current order lines,
  last N chat messages) as its instruction text. Output: AssistantResponse{replyText: String,
  toolName: Optional<String>, reservationArg: Optional<ReservationRequest>,
  orderArg: Optional<List<OrderLineItem>>, respondedAt: Instant}. The agent is configured
  with a before-agent-response guardrail (ResponseGuardrail) and a before-tool-call guardrail
  (ToolCallGuardrail) registered via the agent's guardrail-configuration block.

- 1 Workflow SessionWorkflow per sessionId with steps:
  * agentStep — emits no entity event (message receipt already recorded); calls
    componentClient.forAutonomousAgent(RestaurantAssistantAgent.class, "assistant-" + sessionId)
    .runSingleTask(TaskDef.instructions(formatSessionContext(session, incomingText)))
    — returns a taskId, then forTask(taskId).result(HANDLE_MESSAGE) to fetch the response.
    On success: if response.toolName is empty, calls SessionEntity.recordAgentReply(response)
    and ends. If toolName is present, advances to commitStep.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(SessionWorkflow::error).
  * commitStep — dispatches on toolName: "makeReservation" → calls SessionEntity
    .confirmReservation(reservation); "submitOrder" → calls SessionEntity.commitOrder(order).
    WorkflowSettings.stepTimeout 10s. On success advances to agentStep (ready for next
    message) or closeStep if the session intent is fulfilled.
  * closeStep — calls SessionEntity.close(). WorkflowSettings.stepTimeout 5s.
  * error step — calls SessionEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  messages: List<CustomerMessage>, reservation: Optional<Reservation>,
  order: Optional<Order>, status: SessionStatus, openedAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus enum: OPEN, ACTIVE, RESERVATION_HELD,
  ORDER_PLACED, CLOSED, FAILED. Events: SessionOpened{sessionId, openedAt},
  MessageAdded{message}, AgentReplied{message}, ReservationConfirmed{reservation},
  OrderCommitted{order}, SessionClosed{closedAt}, SessionFailed{reason}. Commands:
  open, addMessage, recordAgentReply, confirmReservation, commitOrder, close, fail,
  getSession. emptyState() returns Session.initial("") with empty message list, all
  Optional fields as Optional.empty(), and status OPEN. emptyState() never references
  commandContext() (Lesson 3). Every Optional<T> field uses Optional.empty() in initial
  state and Optional.of(...) inside the event-applier.

- 1 View SessionView with row type SessionRow (mirrors Session; messages list is truncated
  to the last 20 entries for the view row to keep size bounded). ONE query getAllSessions:
  SELECT * AS sessions FROM session_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {guestName?, contactPhone?}; mints
    sessionId; calls SessionEntity.open; returns {sessionId}), POST /sessions/{id}/messages
    (body {text}; records message; starts or resumes SessionWorkflow; returns {messageId,
    sessionId}), GET /sessions (list from getAllSessions, sorted newest-first), GET
    /sessions/{id} (one row), GET /sessions/sse (Server-Sent Events forwarded from the
    view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SessionTasks.java declaring one Task<R> constant: HANDLE_MESSAGE = Task.name("Handle
  customer message").description("Read the session context and customer message; return
  an AssistantResponse with a reply and optional tool call").resultConformsTo(
  AssistantResponse.class). DO NOT skip this — the AutonomousAgent requires its companion
  Tasks class (Lesson 7).

- Domain records MenuItem, CustomerMessage, ReservationRequest, Reservation, OrderLineItem,
  Order, OrderType, AssistantResponse, Session, SessionStatus.

- ResponseGuardrail.java implementing the before-agent-response hook. Loads the in-process
  menu catalog (from src/main/resources/sample-events/menu-items.jsonl), checks every item
  name cited in replyText exists in the catalog, checks prices match within £0.01, checks
  the reply contains no prohibited phrases. Returns Guardrail.reject(<structured-error>)
  on failure.

- ToolCallGuardrail.java implementing the before-tool-call hook. Dispatches on toolName.
  For makeReservation: asserts date >= now + 1 hour, partySize in [1, 20], time is HH:mm
  in operating hours (12:00-22:00). For submitOrder: asserts every itemId exists in the
  catalog and quantity >= 1. Returns Guardrail.reject(<structured-error>) on failure.

- MenuCatalog.java — an in-process singleton loaded at startup from
  src/main/resources/sample-events/menu-items.jsonl. Exposes findByName(String),
  findById(String), listAll(). Used by ResponseGuardrail, ToolCallGuardrail, and the
  agent's context formatter.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9309 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The RestaurantAssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/menu-items.jsonl with 12 seeded menu items: 3 starters,
  4 mains (including 2 vegetarian, 1 vegan), 2 desserts, 3 drinks. Each item carries full
  dietary flags and allergen list.

- src/main/resources/sample-events/seed-sessions.jsonl with 3 example session transcripts:
  a menu-query session (no write), a reservation session (RESERVATION_HELD), and an order
  session (ORDER_PLACED). Used by the App UI "Load example" button.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root pre-filled for cx-support; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/restaurant-assistant.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Restaurant Assistant", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of session cards; right = selected-session chat thread with bottom-anchored
  input bar and inline reservation/order confirmation cards).
  Browser title exactly: <title>Akka Sample: Restaurant Assistant</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
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
    handle-customer-message.json — 10 AssistantResponse entries:
      3 menu-query replies (no toolName; accurate item references from the seeded catalog),
      3 reservation-flow replies (toolName="makeReservation"; valid ReservationRequest args),
      2 order-flow replies (toolName="submitOrder"; valid OrderLineItem lists referencing
      seeded itemIds),
      2 deliberately INVALID entries: one where replyText cites a menu item not in the
      catalog (triggers ResponseGuardrail), one where makeReservation has a past date
      (triggers ToolCallGuardrail). The mock selects an invalid entry on the FIRST iteration
      of every 4th session (modulo seed) so J4 is reproducible.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. RestaurantAssistantAgent
  extends akka.javasdk.agent.autonomous.AutonomousAgent. The companion SessionTasks.java
  MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (agentStep 60s, commitStep 10s,
  closeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Session is Optional<T>.
- Lesson 7: SessionTasks.java with HANDLE_MESSAGE = Task.name(...).description(...)
  .resultConformsTo(AssistantResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9309 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent
  (RestaurantAssistantAgent). Both guardrails are hooks on the agent — they are NOT
  separate agents.
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
