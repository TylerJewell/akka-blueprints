# SPEC — whatsapp-order-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WhatsApp Order Agent.
**One-line pitch:** A customer sends free-text WhatsApp messages to place, modify, track, or cancel orders; one AI agent reads the product catalog and order database, executes the right action, and replies in natural language — every write is gated by a guardrail, PII is stripped from stored context, and large orders pause for human approval.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `OrderAgent` (AutonomousAgent) carries the entire conversation: it reads product data, decides which tool to call, and replies to the customer. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** intercepts every tool invocation before it executes — so a create-order, modify-order, or cancel-order call is validated (SKU exists, quantity is positive, customer is the order owner) before the write commits. A blocked tool call returns a structured rejection to the agent loop so it can re-plan within its iteration budget.
- A **PII sanitizer** runs inside a Consumer after each conversation turn. It redacts phone numbers, delivery addresses, payment tokens, and customer identifiers from the stored conversation context so the durable log never contains raw contact data. The raw turn is preserved on the entity for audit only.
- An **application-level HITL gate** fires when an order's value exceeds a configurable threshold (default 500 currency units). The workflow transitions the session to `AWAITING_APPROVAL`, a human-review card appears in the UI, and the agent waits. An operator approves or rejects via the API; the agent resumes on approval or notifies the customer on rejection.

The on-decision evaluator for this pattern is a rule-based `TurnScorer` that checks whether the agent's reply references the tool result, is appropriately brief, and does not reveal a redacted field. No second LLM call.

## 3. User-facing flows

The user opens the App UI tab and selects the **Chat** pane, which simulates a WhatsApp message thread.

1. The operator picks a customer session from the dropdown (or starts a new one with a phone number).
2. The operator types a customer message (e.g., "Hi, I'd like to order 3 units of the Blue Widget") and clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/turns` and receives a `turnId`.
3. The session appears in `ACTIVE` state. Within ~1–5 s the agent replies. The reply card shows the agent's message and a small tool-calls panel (collapsed by default) listing which tools fired and whether the guardrail passed each one.
4. If the order total exceeds the HITL threshold, the session transitions to `AWAITING_APPROVAL`. A yellow banner appears: "This order requires operator approval. Review below." The operator can inspect the pending order and click **Approve** or **Reject**.
5. On approval the session resumes, the agent sends the customer a confirmation, and the session transitions to `COMPLETING`.
6. The conversation history panel on the right shows all turns with sanitized context; a PII-categories chip (e.g., `phone`, `address`) appears on any turn where redaction ran.
7. The operator can replay any earlier message or start a fresh session.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `OrderEndpoint` | `HttpEndpoint` | `/api/sessions/*`, `/api/orders/*`, `/api/hitl/*` — turn submission, order ops, HITL approvals, SSE. Serves `/api/metadata/*`. | — | `OrderEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `OrderEntity` | `EventSourcedEntity` | Per-order lifecycle: draft → confirmed → shipped → cancelled. Source of truth. | `OrderEndpoint`, `OrderWorkflow` | `SessionView` |
| `SessionEntity` | `EventSourcedEntity` | Per-session conversation lifecycle: idle → active → awaiting-approval → completing → closed. | `OrderEndpoint`, `ConversationSanitizer`, `OrderWorkflow` | `SessionView` |
| `ConversationSanitizer` | `Consumer` | Subscribes to `TurnCompleted` events; redacts PII from conversation context; calls `SessionEntity.attachSanitizedTurn`. | `SessionEntity` events | `SessionEntity` |
| `OrderWorkflow` | `Workflow` | One workflow per session. Steps: `activateStep` → `agentStep` → `hitlStep` (conditional) → `completeStep`. | started by `OrderEndpoint` on first turn | `OrderAgent`, `OrderEntity`, `SessionEntity` |
| `OrderAgent` | `AutonomousAgent` | The one decision-making LLM. Receives conversation history + current turn; calls product lookup, order create/modify/cancel/status tools; returns a natural-language reply. | invoked by `OrderWorkflow.agentStep` | returns `AgentReply` |
| `ToolGuardrail` | supporting class | Registered on `OrderAgent` as `before-tool-call` hook. Validates every tool invocation before it executes. | `OrderAgent` | passes or rejects |
| `TurnScorer` | supporting class | Rule-based scorer run inside `completeStep`. No LLM call. | `OrderWorkflow` | `TurnEval` |
| `SessionView` | `View` | Read model: one row per session + latest turn for the UI. | `SessionEntity` events, `OrderEntity` events | `OrderEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Product(
    String sku,
    String name,
    String description,
    int stockQuantity,
    double unitPrice,
    String currency
) {}

record OrderLineItem(
    String sku,
    String productName,
    int quantity,
    double unitPrice,
    double lineTotal
) {}

record OrderRequest(
    String orderId,
    String sessionId,
    String customerId,
    List<OrderLineItem> items,
    double orderTotal,
    String currency,
    String deliveryAddress,
    Instant requestedAt
) {}

record AgentReply(
    String replyText,
    List<ToolCall> toolCallsSummary,
    boolean hitlRequired,
    double pendingOrderTotal,
    Instant repliedAt
) {}

record ToolCall(
    String toolName,
    String arguments,    // JSON string
    String outcome,      // "allowed" | "blocked"
    String blockReason   // null when allowed
) {}

record TurnContext(
    String turnId,
    String customerMessage,
    String agentReply,
    List<String> piiCategoriesRedacted
) {}

record TurnEval(
    int score,           // 1..5
    String rationale,
    Instant evaluatedAt
) {}

record Session(
    String sessionId,
    String customerId,
    SessionStatus status,
    List<TurnContext> turns,
    Optional<OrderRequest> pendingOrder,
    Optional<TurnEval> lastTurnEval,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum SessionStatus {
    IDLE, ACTIVE, AWAITING_APPROVAL, COMPLETING, CLOSED, FAILED
}

record Order(
    String orderId,
    String sessionId,
    OrderStatus status,
    Optional<OrderRequest> request,
    Instant createdAt,
    Optional<Instant> confirmedAt,
    Optional<Instant> cancelledAt
) {}

enum OrderStatus {
    DRAFT, CONFIRMED, SHIPPED, CANCELLED, FAILED
}
```

Events on `SessionEntity`: `SessionStarted`, `TurnReceived`, `TurnCompleted`, `ConversationSanitized`, `ApprovalRequested`, `ApprovalGranted`, `ApprovalRejected`, `SessionCompleted`, `SessionFailed`.

Events on `OrderEntity`: `OrderDrafted`, `OrderConfirmed`, `OrderShipped`, `OrderCancelled`, `OrderFailed`.

Every nullable lifecycle field on `Session` and `Order` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ customerId, whatsappPhoneNumber }` → `{ sessionId }`.
- `POST /api/sessions/{id}/turns` — body `{ customerMessage }` → `{ turnId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session with full turn history.
- `GET /api/sessions/sse` — Server-Sent Events; one event per session state transition.
- `POST /api/hitl/{orderId}/approve` — operator approves a pending HITL order.
- `POST /api/hitl/{orderId}/reject` — body `{ reason }` → operator rejects.
- `GET /api/orders` — list all orders.
- `GET /api/orders/{id}` — one order.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: WhatsApp Order Agent</title>`.

The App UI tab is a two-column layout: a left rail with the session list (status pill + customer id + last message + age) and HITL queue (orders awaiting approval); a right pane with the selected session's chat thread — each turn shows the customer message, the agent reply, tool-calls panel (collapsed), PII-categories chip, and turn eval score.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call issued by `OrderAgent`. Validates that (1) the SKU referenced in a create/modify call exists in the product catalog, (2) the requested quantity is a positive integer and does not exceed available stock, (3) any cancel or modify call targets an order owned by the session's `customerId`. On failure returns a structured rejection to the agent loop so it can re-plan within its `maxIterationsPerTask(4)` budget.
- **S1 — PII sanitizer** (`pii`, applied inside `ConversationSanitizer` Consumer): redacts phone numbers, delivery addresses, payment-card-like tokens, and customer-identifier patterns from the `TurnContext.customerMessage` and `TurnContext.agentReply` before the turn is stored in the durable `turns` list on `SessionEntity`. Raw turn text is preserved separately on the entity for audit but is never in the projected `SessionView`.
- **H1 — application HITL gate**: when `AgentReply.hitlRequired == true` (set by the agent when `pendingOrderTotal > HITL_THRESHOLD`), the workflow transitions to `hitlStep` and emits `ApprovalRequested`. The session moves to `AWAITING_APPROVAL`. The workflow polls `SessionEntity` every 2 s; on `ApprovalGranted` it resumes `completeStep`; on `ApprovalRejected` it notifies the agent and closes the session. `HITL_THRESHOLD` is configured in `application.conf` and defaults to `500.0`.

## 9. Agent prompts

- `OrderAgent` → `prompts/order-agent.md`. The single decision-making LLM. System prompt instructs it to serve as a helpful order management assistant, use available tools to fulfill customer requests, and — when an order exceeds the HITL threshold — to set `hitlRequired: true` in its reply payload rather than proceeding.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer sends "order 2 Blue Widgets"; the agent calls create-order, the guardrail passes, and the reply contains a confirmed order number within 15 s.
2. **J2** — Customer requests to cancel a different customer's order; the `before-tool-call` guardrail blocks the cancel tool call with reason `"order-not-owned-by-caller"`; the agent replies that it cannot process that request; no cancel event is emitted.
3. **J3** — Customer sends an order totalling 600 units (above the 500 threshold); the session transitions to `AWAITING_APPROVAL`; after operator approval via `POST /api/hitl/{orderId}/approve`, the agent confirms the order and the session closes.
4. **J4** — A turn where the customer message contains `+1-555-867-5309` and `123 Main St`; the sanitized `TurnContext.customerMessage` shows `[REDACTED-PHONE]` and `[REDACTED-ADDRESS]`; the raw text is retained on the entity but absent from the `SessionView` row.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named whatsapp-order-agent demonstrating the single-agent × cx-support cell.
Runs out of the box (no external WhatsApp API — channel is simulated via the UI). Maven group
io.akka.samples. Maven artifact single-agent-cx-support-whatsapp-order-agent. Java package
io.akka.samples.whatsapporderagent. Akka 3.6.0. HTTP port 9504.

Components to wire (exactly):

- 1 AutonomousAgent OrderAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/order-agent.md>) and
  .capability(TaskAcceptance.of(ORDER_TASKS).maxIterationsPerTask(4)).
  The agent receives the current customer message and conversation history as
  TaskDef.instructions(...) and the product catalog snapshot as a TaskDef.attachment(
  "catalog.json", catalogBytes). Output: AgentReply{replyText: String,
  toolCallsSummary: List<ToolCall>, hitlRequired: boolean, pendingOrderTotal: double,
  repliedAt: Instant}. The agent is configured with a before-tool-call guardrail (see G1
  in eval-matrix.yaml) registered via the agent's guardrail-configuration block.

- 1 Workflow OrderWorkflow per sessionId with four steps:
  * activateStep — emits SessionStarted if first turn, transitions SessionEntity to ACTIVE.
    WorkflowSettings.stepTimeout 5s.
  * agentStep — calls componentClient.forAutonomousAgent(OrderAgent.class,
    "agent-" + sessionId).runSingleTask(TaskDef.instructions(buildTurnPrompt(session))
    .attachment("catalog.json", catalogJsonBytes)) — returns a taskId, then
    forTask(taskId).result(ORDER_TASKS) to fetch AgentReply. Calls
    SessionEntity.recordTurnCompleted(agentReply). If agentReply.hitlRequired() is true,
    advances to hitlStep; otherwise advances to completeStep. WorkflowSettings.stepTimeout
    60s with defaultStepRecovery maxRetries(2).failoverTo(OrderWorkflow::error).
  * hitlStep — emits ApprovalRequested; polls SessionEntity every 2s for ApprovalGranted or
    ApprovalRejected; on ApprovalGranted advances to completeStep; on ApprovalRejected
    transitions entity to FAILED with reason "operator-rejected". WorkflowSettings.stepTimeout
    300s (5-minute operator window).
  * completeStep — runs TurnScorer (deterministic, no LLM) over the AgentReply; emits
    TurnEval via SessionEntity.recordTurnEval(eval); transitions SessionEntity to COMPLETING.
    WorkflowSettings.stepTimeout 5s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 2 EventSourcedEntities:

  * SessionEntity (one per sessionId). State Session{sessionId: String, customerId: String,
    status: SessionStatus, turns: List<TurnContext>, pendingOrder: Optional<OrderRequest>,
    lastTurnEval: Optional<TurnEval>, createdAt: Instant, closedAt: Optional<Instant>}.
    SessionStatus enum: IDLE, ACTIVE, AWAITING_APPROVAL, COMPLETING, CLOSED, FAILED. Events:
    SessionStarted{customerId}, TurnReceived{customerMessage, turnId},
    TurnCompleted{agentReply, turnId}, ConversationSanitized{sanitizedTurn, turnId},
    ApprovalRequested{orderId, orderTotal}, ApprovalGranted{orderId},
    ApprovalRejected{orderId, reason}, SessionCompleted{}, SessionFailed{reason}.
    Commands: startSession, receiveTurn, recordTurnCompleted, attachSanitizedTurn,
    requestApproval, grantApproval, rejectApproval, complete, fail, getSession.
    emptyState() returns Session.initial("") with all Optional fields as Optional.empty()
    and turns as List.of() (Lesson 3). Every Optional<T> field uses Optional.empty() in
    initial state and Optional.of(...) inside the event-applier.

  * OrderEntity (one per orderId). State Order{orderId: String, sessionId: String,
    status: OrderStatus, request: Optional<OrderRequest>, createdAt: Instant,
    confirmedAt: Optional<Instant>, cancelledAt: Optional<Instant>}.
    OrderStatus enum: DRAFT, CONFIRMED, SHIPPED, CANCELLED, FAILED. Events: OrderDrafted{request},
    OrderConfirmed{}, OrderShipped{}, OrderCancelled{reason}, OrderFailed{reason}.
    Commands: draft, confirm, ship, cancel, fail, getOrder.
    emptyState() returns Order.initial("") (Lesson 3).

- 1 Consumer ConversationSanitizer subscribed to SessionEntity TurnCompleted events.
  Runs a regex+heuristic pipeline (phone numbers E.164 and NANP, delivery address patterns,
  payment-card-like tokens, common customer-ID patterns) over the raw customerMessage and
  agentReply fields, builds a sanitized TurnContext with piiCategoriesRedacted list, then
  calls SessionEntity.attachSanitizedTurn(sanitizedTurn, turnId).

- 1 View SessionView with row type SessionRow (mirrors Session minus any raw message fields
  — the view holds only sanitized turns for the UI). Table updater consumes SessionEntity
  and OrderEntity events. ONE query getAllSessions: SELECT * AS sessions FROM session_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * OrderEndpoint at /api with:
    POST /sessions (body {customerId, whatsappPhoneNumber}; mints sessionId; calls
      SessionEntity.startSession; starts OrderWorkflow; returns {sessionId}),
    POST /sessions/{id}/turns (body {customerMessage}; mints turnId; calls
      SessionEntity.receiveTurn; resumes OrderWorkflow.agentStep; returns {turnId}),
    GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{id} (one row),
    GET /sessions/sse (Server-Sent Events forwarded from the view's stream-updates),
    POST /hitl/{orderId}/approve (calls SessionEntity.grantApproval),
    POST /hitl/{orderId}/reject (body {reason}; calls SessionEntity.rejectApproval),
    GET /orders (list all orders),
    GET /orders/{id} (one order),
    GET /api/metadata/* serving YAML/MD files.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- OrderTasks.java declaring one Task<R> constant: ORDER_TASKS = Task.name("Handle customer
  order request").description("Read the customer message and conversation history, call
  appropriate product/order tools, and return an AgentReply").resultConformsTo(AgentReply.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Product, OrderLineItem, OrderRequest, AgentReply, ToolCall, TurnContext,
  TurnEval, Session, SessionStatus, Order, OrderStatus.

- ToolGuardrail.java implementing the before-tool-call hook. Reads the pending tool name
  and arguments from the agent's tool invocation, runs the three checks listed in
  eval-matrix.yaml G1 (SKU exists, quantity valid, order ownership), and either passes
  the invocation through or returns Guardrail.reject(<structured-error>) naming the
  failed check.

- TurnScorer.java — pure deterministic logic (no LLM). Inputs: AgentReply plus the raw
  customerMessage. Outputs: TurnEval. Scoring rubric: (1) reply references the tool outcome
  (+2), (2) reply length appropriate for the request type (+1), (3) no redacted-field marker
  in agentReply (+1), (4) hitlRequired flag consistent with order total (+1). Score 1–5.

- ProductCatalogService.java — in-process product catalog loaded from
  src/main/resources/sample-events/products.jsonl at startup. Exposes lookupBySku(sku),
  listAll(), checkStock(sku, quantity). Used by OrderWorkflow to build the catalog attachment
  and by ToolGuardrail to validate SKUs.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9504 and
  hitl.order.threshold = 500.0; the three model-provider blocks (anthropic claude-sonnet-4-6,
  openai gpt-4o, googleai-gemini gemini-2.5-flash) reading the canonical env vars
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The
  OrderAgent.definition() binds the configured provider.

- src/main/resources/sample-events/products.jsonl with 8 seeded products (electronics,
  apparel, home goods — mix of low-value and high-value items so J1/J3 are exercisable
  without editing the catalog).

- src/main/resources/sample-events/seed-sessions.jsonl with 3 seeded customer sessions
  showing different conversation flows: a simple 1-item order, a multi-item order above
  the HITL threshold, and a cancel request.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, S1, H1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = true, sector = retail-cx,
  decisions.authority_level = autonomous-write (writes execute after guardrail passes),
  oversight.human_on_loop = true (HITL gate for high-value orders).

- prompts/order-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: WhatsApp Order Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  session list + HITL queue; right = selected session's chat thread with tool-calls panel,
  PII chips, and turn eval score). Browser title exactly:
  <title>Akka Sample: WhatsApp Order Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    handle-customer-order-request.json — 8 AgentReply entries covering a variety of
    orders: single-item order under threshold (hitlRequired=false), multi-item order
    over threshold (hitlRequired=true), status check, modification, cancellation, product
    lookup only, guardrail-blocked attempt (toolCallsSummary shows one blocked call
    followed by a re-planned allowed call), and an ambiguous message triggering a
    clarifying reply. Each entry has a realistic replyText, toolCallsSummary list, and
    pendingOrderTotal. Plus 2 deliberately malformed entries (missing replyText, invalid
    pendingOrderTotal type) to exercise schema validation.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. OrderAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion OrderTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (activateStep 5s, agentStep 60s,
  hitlStep 300s, completeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Session and Order is Optional<T>.
- Lesson 7: OrderTasks.java with ORDER_TASKS = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9504 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (OrderAgent). TurnScorer is
  deterministic and does NOT make an LLM call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external post-hoc check.
- The product catalog is passed as a Task ATTACHMENT, not inlined into the agent's
  instructions text.
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
