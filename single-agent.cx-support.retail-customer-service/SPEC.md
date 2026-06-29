# SPEC — retail-customer-service

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** RetailCS.
**One-line pitch:** A customer sends a message about a plant; one AI agent looks up the product catalog and the customer's orders, answers the question or applies the requested order change, and returns a structured reply — all while two guardrails ensure order modifications are state-valid and every response is appropriate for a public channel.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `CustomerServiceAgent` (AutonomousAgent) carries the entire customer interaction; surrounding components manage session state, order data, and governance checks. Two guardrails are wired around the agent:

- A **before-tool-call guardrail** fires before any order-modification tool (cancel, update-address, update-quantity) is executed. It validates that the current order status permits the requested modification — an order that has already shipped cannot be cancelled; a delivered order cannot have its address changed. The guardrail returns a structured rejection that the agent incorporates into its reply.
- A **before-agent-response guardrail** runs on every outbound reply before the message reaches the customer. It checks for policy violations: competitor brand mentions, pricing commitments outside allowed bands, and discouraging or dismissive language. A rejected response causes the agent to rewrite on the next iteration.

The blueprint shows that even a stateless conversational pattern needs governance at both the tool boundary (data integrity) and the output boundary (brand and policy safety).

## 3. User-facing flows

The user opens the App UI tab.

1. The customer types a message into the **Message** input. Optionally picks an order from the **Order context** dropdown (which seeded examples include).
2. The customer clicks **Send**. The UI POSTs to `/api/sessions` (first turn) or `/api/sessions/{id}/turn` (subsequent turns) and receives a `sessionId` or the updated session.
3. The session card appears in the live list in `OPEN` state. Within ~1–30 s, the agent's reply appears in `REPLIED` state on the card.
4. If the message requested an order change and the guardrail allowed it, the right pane shows an **Order modification** chip alongside the reply with the change that was applied.
5. If the guardrail blocked the tool call, the agent's reply explains the limitation; the right pane shows a **Guardrail block** chip with the rejection reason.
6. The customer can continue the conversation; the live list keeps the full session history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CustomerServiceEndpoint` | `HttpEndpoint` | `/api/sessions/*` — start session, send turn, list, get, SSE; `/api/products/*` — catalog lookup; `/api/orders/*` — order lookup; `/api/metadata/*`. | — | `SessionEntity`, `OrderEntity`, `SessionView`, `OrderView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session conversation state: open → active → closed. Holds turn history. | `CustomerServiceEndpoint`, `SessionWorkflow` | `SessionView` |
| `OrderEntity` | `EventSourcedEntity` | Per-order state: pending → processing → shipped → delivered / cancelled. Holds line items and modification log. | `CustomerServiceEndpoint`, `CustomerServiceAgent` tool calls | `OrderView` |
| `SessionWorkflow` | `Workflow` | One workflow per session turn. Steps: `agentStep` (invoke agent) → `applyModificationStep` (apply any order change) → `replyStep` (write reply to entity). | started by `CustomerServiceEndpoint` on each turn | `CustomerServiceAgent`, `OrderEntity`, `SessionEntity` |
| `CustomerServiceAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the conversation history and product/order context as a task attachment; returns `AgentReply`. Two guardrails registered: `OrderModificationGuardrail` (before-tool-call) and `ReplyPolicyGuardrail` (before-agent-response). | invoked by `SessionWorkflow` | returns `AgentReply` |
| `OrderModificationGuardrail` | guardrail class | `before-tool-call` on `cancelOrder`, `updateOrderAddress`, `updateOrderQuantity`. Validates the order's current status permits the modification. | bound to `CustomerServiceAgent` | pass-through or structured rejection |
| `ReplyPolicyGuardrail` | guardrail class | `before-agent-response` on all outbound replies. Checks competitor mentions, pricing commitments, and tone policy. | bound to `CustomerServiceAgent` | pass-through or structured rejection |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `CustomerServiceEndpoint` |
| `OrderView` | `View` | Read model: one row per order. | `OrderEntity` events | `CustomerServiceEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Product(
    String productId,
    String name,
    String category,
    String description,
    double priceUsd,
    boolean inStock,
    String careInstructions
) {}

record OrderLineItem(
    String productId,
    String productName,
    int quantity,
    double unitPriceUsd
) {}

record Order(
    String orderId,
    String customerId,
    List<OrderLineItem> lineItems,
    String shippingAddress,
    OrderStatus status,
    Instant placedAt,
    Optional<Instant> shippedAt,
    Optional<Instant> deliveredAt,
    List<OrderModification> modifications
) {}

enum OrderStatus { PENDING, PROCESSING, SHIPPED, DELIVERED, CANCELLED }

record OrderModification(
    String modificationType,   // "cancel" | "update-address" | "update-quantity"
    String field,
    String previousValue,
    String newValue,
    Instant appliedAt
) {}

record ConversationTurn(
    String turnId,
    String customerMessage,
    Optional<AgentReply> reply,
    Optional<String> orderContext,    // orderId if the turn was order-scoped
    TurnStatus status,
    Instant startedAt,
    Optional<Instant> completedAt
) {}

enum TurnStatus { PENDING, REPLIED, GUARDRAIL_BLOCKED, FAILED }

record AgentReply(
    String message,
    Optional<OrderChangeRequest> orderChange,
    String model,
    Instant repliedAt
) {}

record OrderChangeRequest(
    String orderId,
    String changeType,        // "cancel" | "update-address" | "update-quantity"
    String field,
    String newValue
) {}

record Session(
    String sessionId,
    String customerId,
    List<ConversationTurn> turns,
    SessionStatus status,
    Instant openedAt,
    Optional<Instant> closedAt
) {}

enum SessionStatus { OPEN, ACTIVE, CLOSED }
```

Events on `SessionEntity`: `SessionOpened`, `TurnStarted`, `TurnReplied`, `TurnBlockedByGuardrail`, `TurnFailed`, `SessionClosed`.
Events on `OrderEntity`: `OrderPlaced`, `OrderProcessing`, `OrderShipped`, `OrderDelivered`, `OrderCancelled`, `OrderAddressUpdated`, `OrderQuantityUpdated`.

Every nullable lifecycle field uses `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ customerId, customerMessage, orderId? }` → `{ sessionId }`. Starts a new session and first turn.
- `POST /api/sessions/{id}/turn` — body `{ customerMessage, orderId? }` → `{ turnId }`. Continues an existing session.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session with all turns.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/orders/{id}` — one order (for agent tool calls + UI drill-down).
- `GET /api/products` — full product list from seeded catalog.
- `GET /api/products/{id}` — one product.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: RetailCS</title>`.

The App UI tab is a two-column layout: a left rail with active and recent sessions (status pill + age + customer id) and a right pane with the selected session's conversation thread — customer messages and agent replies in chronological order, with order-context chips, guardrail-block indicators, and an order-change confirmation card for modifications that succeeded.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`OrderModificationGuardrail`): fires before any of the three order-mutation tool calls. Fetches the current `OrderStatus` from `OrderEntity`. Rejects with a structured code (`ORDER_NOT_MODIFIABLE`) if the requested modification is not allowed for the current status (e.g., cannot cancel a `SHIPPED` or `DELIVERED` order; cannot update address after `PROCESSING`). The agent receives the rejection and explains it to the customer without exposing internal detail.
- **G2 — before-agent-response guardrail** (`ReplyPolicyGuardrail`): runs on every candidate reply before it leaves the agent loop. Checks that the message does not mention competitor brand names, does not quote a price more than 10% below the catalog price, and does not contain dismissive or rude language patterns. On failure, returns a structured rejection with the policy code that failed; the agent loop retries within its iteration budget.

## 9. Agent prompts

- `CustomerServiceAgent` → `prompts/customer-service-agent.md`. The single decision-making LLM. System prompt defines the agent's persona, available tools, behavior rules, and how to handle guardrail rejections gracefully.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer asks how to care for a succulent from the catalog; agent looks up the product and returns accurate care instructions grounded in the catalog entry.
2. **J2** — Customer requests an address change on a `PENDING` order; the before-tool-call guardrail allows it; the modification is applied; the confirmation appears in the session.
3. **J3** — Customer requests cancellation of a `SHIPPED` order; the before-tool-call guardrail blocks it; the agent explains the policy without revealing the rejection code; no `OrderCancelled` event is emitted.
4. **J4** — Agent's first draft reply mentions a competitor by name; the before-agent-response guardrail rejects it; the second iteration produces a compliant reply; the UI never displays the rejected draft.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named retail-customer-service demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-retail-customer-service. Java package io.akka.samples.customerservice.
Akka 3.6.0. HTTP port 9523.

Components to wire (exactly):

- 1 AutonomousAgent CustomerServiceAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/customer-service-agent.md>) and
  .capability(TaskAcceptance.of(CustomerServiceTasks.HANDLE_CUSTOMER_TURN)
    .maxIterationsPerTask(4)). The task receives the conversation history and product/order
  context as a task ATTACHMENT named "context.json" (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output: AgentReply{message:
  String, orderChange: Optional<OrderChangeRequest>, model: String, repliedAt: Instant}.
  The agent is configured with two guardrails: OrderModificationGuardrail bound to
  before-tool-call on tools ["cancelOrder","updateOrderAddress","updateOrderQuantity"], and
  ReplyPolicyGuardrail bound to before-agent-response. Both registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  4-iteration budget.

- 1 Workflow SessionWorkflow per session turn (id = "turn-" + sessionId + "-" + turnId)
  with three steps:
  * agentStep — emits TurnStarted, then calls componentClient.forAutonomousAgent(
    CustomerServiceAgent.class, "cs-" + sessionId).runSingleTask(
      TaskDef.instructions("Handle the customer message.")
        .attachment("context.json", buildContextJson(session, products, orders).getBytes())
    ) — returns a taskId, then forTask(taskId).result(HANDLE_CUSTOMER_TURN) to fetch the
    AgentReply. WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(SessionWorkflow::error).
  * applyModificationStep — if agentReply.orderChange().isPresent(), calls the appropriate
    OrderEntity command (cancelOrder / updateOrderAddress / updateOrderQuantity). Idempotent:
    if the command returns a conflict (order already in terminal state), marks the turn as
    GUARDRAIL_BLOCKED and skips the replyStep. WorkflowSettings.stepTimeout 15s.
  * replyStep — calls SessionEntity.recordTurnReply(turnId, agentReply). WorkflowSettings
    .stepTimeout 5s. error step transitions the entity to FAILED via SessionEntity.failTurn.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  customerId: String, turns: List<ConversationTurn>, status: SessionStatus, openedAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus enum: OPEN, ACTIVE, CLOSED. Events: SessionOpened,
  TurnStarted, TurnReplied, TurnBlockedByGuardrail, TurnFailed, SessionClosed. Commands:
  openSession, startTurn, recordTurnReply, blockTurn, failTurn, closeSession, getSession.
  emptyState() returns Session.initial("") with empty turns list and no commandContext()
  reference (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state.

- 1 EventSourcedEntity OrderEntity (one per orderId). State Order{orderId: String,
  customerId: String, lineItems: List<OrderLineItem>, shippingAddress: String, status: OrderStatus,
  placedAt: Instant, shippedAt: Optional<Instant>, deliveredAt: Optional<Instant>,
  modifications: List<OrderModification>}. OrderStatus enum: PENDING, PROCESSING, SHIPPED,
  DELIVERED, CANCELLED. Events: OrderPlaced, OrderProcessing, OrderShipped, OrderDelivered,
  OrderCancelled, OrderAddressUpdated, OrderQuantityUpdated. Commands: placeOrder, markProcessing,
  markShipped, markDelivered, cancelOrder, updateOrderAddress, updateOrderQuantity, getOrder.
  Modification commands reject (return an error Effect) if the order is in a terminal or
  non-modifiable state — this is the second-line defense behind the guardrail.

- 2 Views:
  * SessionView with row type SessionRow (mirrors Session, summarized: includes last-turn
    status and last agent reply preview, excludes full turn list — the full list fetched via
    GET /api/sessions/{id}). ONE query getAllSessions: SELECT * AS sessions FROM session_view.
    No WHERE clause (Lesson 2).
  * OrderView with row type OrderRow (mirrors Order). ONE query getOrder: SELECT * AS orders
    FROM order_view WHERE orderId = :orderId. One query getAllOrders: SELECT * AS orders FROM
    order_view.

- 2 HttpEndpoints:
  * CustomerServiceEndpoint at /api:
    POST /sessions (body {customerId, customerMessage, orderId?}; mints sessionId and turnId;
      calls SessionEntity.openSession then starts SessionWorkflow; returns {sessionId}),
    POST /sessions/{id}/turn (body {customerMessage, orderId?}; mints turnId; starts new
      SessionWorkflow turn; returns {turnId}),
    GET /sessions (list from getAllSessions, newest-first),
    GET /sessions/{id} (one session via getSession command),
    GET /sessions/sse (SSE from view stream-updates),
    GET /orders/{id} (OrderView query),
    GET /products (serves seeded products.jsonl deserialized to List<Product>),
    GET /products/{id} (one product by productId),
    and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- CustomerServiceTasks.java declaring one Task<R> constant: HANDLE_CUSTOMER_TURN =
  Task.name("Handle customer turn").description("Read the customer message and context
  attachment, answer the question or process the order change, and return an AgentReply")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records Product, OrderLineItem, Order, OrderStatus, OrderModification,
  ConversationTurn, TurnStatus, AgentReply, OrderChangeRequest, Session, SessionStatus.

- OrderModificationGuardrail.java implementing the before-tool-call hook on tools
  ["cancelOrder","updateOrderAddress","updateOrderQuantity"]. Fetches current OrderEntity state
  via componentClient.forEventSourcedEntity(OrderEntity.class, orderId).call(getOrder).
  Rejects with Guardrail.reject(OrderModificationGuardrail.RejectionCode.ORDER_NOT_MODIFIABLE
  + " current status: " + order.status()) if status is not in the set of statuses that
  permit the requested modification (cancel: only PENDING; update-address: PENDING or
  PROCESSING; update-quantity: PENDING). Passes through otherwise.

- ReplyPolicyGuardrail.java implementing the before-agent-response hook on all replies.
  Checks: (1) no competitor brand names from a configurable list loaded from
  src/main/resources/policy/competitor-brands.txt; (2) no price quoted more than 10% below
  the catalog price for any product mentioned (requires a ProductCatalog helper); (3) no
  phrases matching a rude-language pattern list from policy/rude-patterns.txt. On any
  failure returns Guardrail.reject(<structured-error>) naming the violated policy code.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9523 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/products.jsonl with 12 seeded Cymbal Home & Garden
  products (succulents, ferns, orchids, herbs, and garden tools) — each with productId,
  name, category, description, priceUsd (5.00–49.99), inStock, and careInstructions.

- src/main/resources/sample-events/orders.jsonl with 5 seeded orders across OrderStatus
  values: one PENDING, one PROCESSING, one SHIPPED, one DELIVERED, one CANCELLED.
  Each references productIds from products.jsonl.

- src/main/resources/policy/competitor-brands.txt — one brand name per line (fictional names
  only in the seed: "GreenThumb Co", "PlantWorld", "LeafyLane").

- src/main/resources/policy/rude-patterns.txt — simple phrase patterns (e.g., "we can't
  help", "not our problem", "that's your fault").

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching Section 8.
  Matching simplified_view list.

- risk-survey.yaml at the project root with cx-support pre-fills per Section 12.

- prompts/customer-service-agent.md loaded as the agent system prompt.

- README.md at project root: title "Akka Sample: RetailCS". NO Configuration section.
  NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs. App UI tab uses a two-column layout (left = session list; right =
  conversation thread with order-context chips and guardrail-block indicators).
  Browser title exactly: <title>Akka Sample: RetailCS</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct AgentReply outputs per task (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface. Dispatches on
  HANDLE_CUSTOMER_TURN. Reads src/main/resources/mock-responses/handle-customer-turn.json.
  Picks one entry pseudo-randomly per call (seedFor(sessionId + turnId)).
- handle-customer-turn.json — 10 AgentReply entries covering: product inquiry (no order
  change), address-change request on a PENDING order (orderChange present), cancel request
  on a PENDING order (orderChange present), cancel request on a SHIPPED order (guardrail
  will block — the mock returns an orderChange with type "cancel"; the guardrail fires and
  rejects it), a reply that mentions a competitor by name (the guardrail will block it
  triggering a retry), and 5 normal informational replies. The guardrail-triggering entries
  exercise both the before-tool-call and before-agent-response paths.
- MockModelProvider.seedFor(sessionId, turnId) makes selection deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: CustomerServiceAgent extends akka.javasdk.agent.autonomous.AutonomousAgent.
  CustomerServiceTasks.java MUST exist with HANDLE_CUSTOMER_TURN.
- Lesson 4: agentStep 60s, applyModificationStep 15s, replyStep 5s, error 5s.
- Lesson 6: all Optional<T> fields use Optional.empty() in initial state.
- Lesson 7: CustomerServiceTasks.java is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9523 declared in application.conf.
- Lesson 11: no source-platform metadata user-facing.
- Lesson 12: UI fits 1080px content column.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and themeVariables.
- Lesson 25: API-key sourcing — never write the key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute; exactly five <section
  class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (CustomerServiceAgent). Both
  guardrails are logic classes, not agents.
- Customer context (order history, product data) is passed as a task ATTACHMENT, not
  inlined into instructions.
- Both guardrails are wired via the agent's guardrail-configuration block, not as external
  post-processing checks.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
