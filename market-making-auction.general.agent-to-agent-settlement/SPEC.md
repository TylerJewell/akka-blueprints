# SPEC — agent-to-agent-settlement

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Agent-to-Agent Commerce Settlement.
**One-line pitch:** Autonomous buyer agents discover service listings, submit bids, and settle the winning transaction — a precondition guardrail gates payment and disputed orders escalate to a human operator.

## 2. What this blueprint demonstrates

The **market-making-auction** coordination pattern wired with Akka's first-party primitives: a Workflow closes an auction, fans settlement work through a precondition guardrail on a SettlementAgent, and persists the result on an OrderEntity. The blueprint also demonstrates a **before-settlement guardrail** that verifies payment conditions before any funds transfer, and a **human-in-the-loop** path that escalates disputed orders to a live operator.

## 3. User-facing flows

The user opens the App UI tab and publishes a service listing via the form, or waits for `ListingSimulator` to drip a listing.

1. The system creates a `ServiceListing` in `OPEN` status and starts an `AuctionWorkflow`.
2. `BuyerAgent` receives the listing, evaluates it, and submits a `Bid` with an offered price.
3. After the auction window (30 seconds in dev), the workflow closes the auction, selects the best bid, and creates an `Order` in `PENDING`.
4. `SettlementAgent` runs the precondition check: verifies counterparty credentials, balance sufficiency, and listing integrity. It returns a `PreconditionReport`.
5. If the report passes, the workflow settles the order (`SETTLED`) and emits a `SettlementRef`. If it fails, the order moves to `CANCELLED` with a `failureReason`.
6. If a settled order is later disputed by the buyer, the order moves to `DISPUTED`. `DisputeMonitor` detects the transition and writes an `OperatorTask` to the operator queue. An operator resolves via `POST /api/orders/{id}/resolve`.

A `ListingSimulator` (TimedAction) drips a sample listing every 60 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BuyerAgent` | `AutonomousAgent` | Evaluates a service listing and submits a bid with a reasoned price. | `AuctionWorkflow` | returns typed result to workflow |
| `SettlementAgent` | `AutonomousAgent` | Verifies settlement preconditions; returns a pass/fail `PreconditionReport`. | `AuctionWorkflow` | returns typed result to workflow |
| `AuctionWorkflow` | `Workflow` | Runs auction close, precondition check, settlement, and dispute gating. | `MarketplaceEndpoint`, `AuctionConsumer` | `ServiceListingEntity`, `OrderEntity` |
| `ServiceListingEntity` | `EventSourcedEntity` | Holds the listing lifecycle (`OPEN → CLOSED → CANCELLED`). | `AuctionWorkflow` | `MarketplaceView` |
| `OrderEntity` | `EventSourcedEntity` | Holds the order lifecycle (`PENDING → PRECHECKED → SETTLED / DISPUTED / CANCELLED`). | `AuctionWorkflow` | `MarketplaceView` |
| `OperatorTaskEntity` | `EventSourcedEntity` | Queue of operator tasks awaiting human resolution. | `DisputeMonitor` | `MarketplaceView` |
| `MarketplaceView` | `View` | Combined read model: active listings, orders, and open operator tasks. | `ServiceListingEntity`, `OrderEntity`, `OperatorTaskEntity` events | `MarketplaceEndpoint` |
| `AuctionConsumer` | `Consumer` | Listens to `ListingPublished` events and starts one `AuctionWorkflow` per listing. | `ServiceListingEntity` events | `AuctionWorkflow` |
| `ListingSimulator` | `TimedAction` | Drips a sample listing every 60 s. | scheduler | `ServiceListingEntity` |
| `DisputeMonitor` | `TimedAction` | Scans for `DISPUTED` orders every 5 minutes; writes an `OperatorTask` if none exists. | scheduler | `OperatorTaskEntity` |
| `MarketplaceEndpoint` | `HttpEndpoint` | `/api/*` — listings, orders, bids, operator resolution, SSE. | — | `MarketplaceView`, `ServiceListingEntity`, `OrderEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record ServiceListing(String listingId, String serviceType, String description,
                      java.math.BigDecimal askPrice, String sellerAgentId,
                      Instant publishedAt) {}

record Bid(String bidId, String listingId, String buyerAgentId,
           java.math.BigDecimal amount, String rationale, Instant placedAt) {}

record PreconditionReport(boolean passed, List<String> checkedConditions,
                          Optional<String> failureReason, Instant checkedAt) {}

record SettlementRef(String ref, Instant settledAt) {}

record Order(
    String orderId,
    String listingId,
    String buyerAgentId,
    String sellerAgentId,
    java.math.BigDecimal agreedPrice,
    OrderStatus status,
    Optional<PreconditionReport> preconditionReport,
    Optional<SettlementRef> settlementRef,
    Optional<String> failureReason,
    Optional<String> disputeReason,
    Optional<String> operatorNote,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum OrderStatus { PENDING, PRECHECKED, SETTLED, DISPUTED, CANCELLED }

record OperatorTask(String taskId, String orderId, String reason,
                    boolean resolved, Instant createdAt,
                    Optional<String> resolution, Optional<Instant> resolvedAt) {}
```

### Events (on `OrderEntity`)

`OrderCreated`, `PreconditionChecked`, `OrderSettled`, `OrderDisputed`, `OrderCancelled`, `DisputeResolved`.

### Events (on `ServiceListingEntity`)

`ListingPublished { listingId, serviceType, description, askPrice, sellerAgentId, publishedAt }`,
`BidReceived { bidId, buyerAgentId, amount, rationale, placedAt }`,
`AuctionClosed { winningBidId, closedAt }`,
`ListingCancelled { reason, cancelledAt }`.

### Events (on `OperatorTaskEntity`)

`TaskCreated { taskId, orderId, reason, createdAt }`,
`TaskResolved { resolution, resolvedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/listings` — body `{ serviceType, description, askPrice, sellerAgentId }` → `{ listingId }`.
- `GET /api/listings` — list all listings. Optional `?status=OPEN|CLOSED|CANCELLED`.
- `GET /api/listings/{id}` — one listing with its bid history.
- `POST /api/listings/{id}/bids` — body `{ buyerAgentId, amount, rationale }` → `{ bidId }`.
- `GET /api/orders` — list all orders. Optional `?status=PENDING|PRECHECKED|SETTLED|DISPUTED|CANCELLED`.
- `GET /api/orders/{id}` — one order.
- `POST /api/orders/{id}/dispute` — body `{ reason }` → `{ orderId }`. Moves order to `DISPUTED`.
- `POST /api/orders/{id}/resolve` — body `{ operatorNote }` → `{ orderId }`. Operator resolves a dispute.
- `GET /api/operator/tasks` — list open operator tasks.
- `GET /api/marketplace/sse` — server-sent events stream of every listing and order change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Agent-to-Agent Commerce Settlement"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to publish a listing, live list of listings and orders streamed via SSE; expand a listing to see its bids; expand an order to see its precondition report, settlement ref, and any dispute / operator resolution.

Browser title: `<title>Akka Sample: Agent-to-Agent Commerce Settlement</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-settlement guardrail** (`before-settlement` on `SettlementAgent`): verifies counterparty credentials, balance sufficiency, and listing integrity before any settlement is authorised. Blocking. Failure → `CANCELLED`.
- **H1 — HITL dispute escalation** (`hitl · application`): `DisputeMonitor` (TimedAction) scans for `DISPUTED` orders every 5 minutes and writes an `OperatorTask` to `OperatorTaskEntity`. The operator resolves via `POST /api/orders/{id}/resolve`.

## 9. Agent prompts

- `BuyerAgent` → `prompts/buyer-agent.md`. Evaluates listings and sets a bid price.
- `SettlementAgent` → `prompts/settlement-agent.md`. Checks settlement preconditions; returns `PreconditionReport`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Publish a listing; `BuyerAgent` bids; auction closes; precondition check passes; order reaches `SETTLED` within 90 s; UI reflects each transition via SSE.
2. **J2** — Inject a failing precondition (SettlementAgent returns `passed: false`); order reaches `CANCELLED` with a `failureReason`.
3. **J3** — Dispute a settled order; `DisputeMonitor` creates an `OperatorTask`; operator resolves it via the API; order gains an `operatorNote`.
4. **J4** — Leave the service running; `ListingSimulator` drips a listing every 60 s; the App UI is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named agent-to-agent-settlement demonstrating the
market-making-auction × general cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
market-making-auction-general-agent-to-agent-settlement.
Java package io.akka.samples.agenttoagentcommercesettlement. Akka 3.6.0. HTTP port 9301.

Components to wire (exactly):
- 2 AutonomousAgents:
  * BuyerAgent — definition() with capability(TaskAcceptance.of(BID).maxIterationsPerTask(2)).
    System prompt loaded from prompts/buyer-agent.md. Returns
    Bid{bidId, listingId, buyerAgentId, amount, rationale, placedAt}.
  * SettlementAgent — capability(TaskAcceptance.of(CHECK_PRECONDITIONS).maxIterationsPerTask(2)).
    System prompt from prompts/settlement-agent.md. Returns
    PreconditionReport{passed, checkedConditions: List<String>, failureReason: Optional<String>,
    checkedAt}.

- 1 Workflow AuctionWorkflow with steps:
  bidStep -> closeAuctionStep -> preconditionStep -> settleStep | cancelStep.
  bidStep calls forAutonomousAgent(BuyerAgent.class, BID) with the ServiceListing payload;
  give bidStep a 60s stepTimeout (Lesson 4).
  closeAuctionStep selects the winning Bid (highest amount) and creates an Order via
  OrderEntity.createOrder; give it a 10s stepTimeout.
  preconditionStep calls forAutonomousAgent(SettlementAgent.class, CHECK_PRECONDITIONS) with the
  Order and ServiceListing; give it a 60s stepTimeout.
  If PreconditionReport.passed is true, transition to settleStep: call OrderEntity.settleOrder,
  emit SettlementRef, end with OrderSettled.
  If PreconditionReport.passed is false, transition to cancelStep: call
  OrderEntity.cancelOrder(failureReason), end with OrderCancelled.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity ServiceListingEntity holding state with fields
  {listingId, serviceType, description, askPrice, sellerAgentId, ListingStatus, List<Bid> bids,
  Optional<String> winningBidId, Instant publishedAt, Optional<Instant> closedAt}.
  ListingStatus enum: OPEN, CLOSED, CANCELLED.
  Events: ListingPublished, BidReceived, AuctionClosed, ListingCancelled.
  Commands: publishListing, receiveBid, closeAuction, cancelListing, getListing.

- 1 EventSourcedEntity OrderEntity holding state Order{orderId, listingId, buyerAgentId,
  sellerAgentId, agreedPrice, OrderStatus, Optional<PreconditionReport> preconditionReport,
  Optional<SettlementRef> settlementRef, Optional<String> failureReason,
  Optional<String> disputeReason, Optional<String> operatorNote,
  Instant createdAt, Optional<Instant> finishedAt}.
  OrderStatus enum: PENDING, PRECHECKED, SETTLED, DISPUTED, CANCELLED.
  Events: OrderCreated, PreconditionChecked, OrderSettled, OrderDisputed, OrderCancelled,
  DisputeResolved.
  Commands: createOrder, setPreconditionResult, settleOrder, disputeOrder, cancelOrder,
  resolveDispute, getOrder.
  emptyState() returns Order.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity OperatorTaskEntity with state
  OperatorTask{taskId, orderId, reason, boolean resolved, Instant createdAt,
  Optional<String> resolution, Optional<Instant> resolvedAt}.
  Events: TaskCreated, TaskResolved.
  Commands: createTask, resolveTask, getTask.

- 1 View MarketplaceView with two query methods:
  getAllListings SELECT * AS listings FROM listing_view and
  getAllOrders SELECT * AS orders FROM order_view.
  Row types: ListingRow (mirrors ServiceListingEntity state, Optional<T> on every nullable field),
  OrderRow (mirrors Order, Optional<T> on every nullable field).
  ONE separate query for operator tasks: getOpenTasks SELECT * AS tasks FROM operator_task_view
  WHERE resolved = false. No WHERE status filter on listings/orders (Akka cannot auto-index enum
  columns) — caller filters client-side.

- 1 Consumer AuctionConsumer subscribed to ServiceListingEntity events; on ListingPublished
  starts an AuctionWorkflow with the listingId as the workflow id.

- 2 TimedActions:
  * ListingSimulator — every 60s, reads next line from
    src/main/resources/sample-events/sample-listings.jsonl and calls
    ServiceListingEntity.publishListing.
  * DisputeMonitor — every 5 minutes, queries MarketplaceView.getAllOrders, picks all
    DISPUTED orders whose orderId has no open OperatorTask (checks getOpenTasks),
    and calls OperatorTaskEntity.createTask for each.

- 2 HttpEndpoints:
  * MarketplaceEndpoint at /api with POST /listings, GET /listings, GET /listings/{id},
    POST /listings/{id}/bids, GET /orders, GET /orders/{id},
    POST /orders/{id}/dispute, POST /orders/{id}/resolve, GET /operator/tasks,
    GET /marketplace/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- CommerceTasks.java declaring two Task<R> constants: BID (Bid), CHECK_PRECONDITIONS
  (PreconditionReport).
- Domain records ServiceListing, Bid, PreconditionReport, SettlementRef, Order, OperatorTask.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9301 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/sample-listings.jsonl with 8 canned listing lines covering
  varied serviceType values (e.g., "data-enrichment", "translation", "image-captioning",
  "sentiment-analysis", "code-review", "summarisation", "fraud-screening", "entity-resolution").
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 before-settlement guardrail,
  H1 hitl application) and a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function =
  agent-to-agent-commerce, decisions.authority_level = automated-within-limits,
  data.data_classes.pii = false, capabilities.* = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/buyer-agent.md, prompts/settlement-agent.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Agent-to-Agent Commerce Settlement",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (listing publish form + live
  list of listings and orders with status pills; expand listing to see bids; expand order to see
  precondition report, settlement ref, dispute info). Browser title exactly:
  <title>Akka Sample: Agent-to-Agent Commerce Settlement</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka
  records only the REFERENCE (env-var name, file path, secrets URI); the
  value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with a per-agent dispatch on the agent class
  name (or the Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (buyer-agent.json,
  settlement-agent.json), picks one entry pseudo-randomly per call,
  and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    buyer-agent.json — 4–6 Bid entries, each with a plausible amount
      (between 0.80× and 1.10× the listing's askPrice), a one-sentence
      rationale for the offered price, and a fresh bidId + placedAt.
    settlement-agent.json — 5 PreconditionReport entries: 4 with
      passed=true (checkedConditions listing credential-check,
      balance-check, listing-integrity) and 1 with passed=false
      (failureReason = "Insufficient balance for escrow").
- A MockModelProvider.seedFor(listingId) helper makes the selection
  deterministic per listing id so the same listing produces the same
  output across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s for bid
  and precondition steps, 10s for close step); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion CommerceTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9301 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and arrow
  labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps (if added) use CompletionStage zip, NOT sequential calls.
- The Overview Try-it card shows just "/akka:build" — no env-var export block.
- No forbidden words in user-facing text: shape, minimal, smaller, complex, Akka SDK in
  narrative, T1/T2/T3/T4, deferred, marketing tone.
```


## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing options from Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
