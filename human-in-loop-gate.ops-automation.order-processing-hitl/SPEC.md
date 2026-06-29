# SPEC — order-processing

The natural-language brief `/akka:specify` reads to generate this system. The whole file — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** HITL Order Processing.
**One-line pitch:** A client submits an order; `ValidationAgent` reviews and prices it; the workflow pauses at an approval task; a human approves or rejects through the API; on approval `FulfillmentAgent` dispatches and returns a tracking id.

## 2. What this blueprint demonstrates

The human-in-loop-gate coordination pattern in the ops-automation domain: a 3-task graph that validates, then waits at an unassigned approval task that a person completes through the API, then fulfills only if approved. The governance pattern is an application-level human approval gate between the validation and fulfillment phases, a before-tool-call guardrail that blocks the fulfillment side-effects unless the order is approved, and a before-agent-response guardrail that checks the validation output before it reaches the reviewer.

## 3. User-facing flows

1. A client POSTs an order to `/api/orders`. The response returns `{ orderId }`. The order appears in the UI in `PENDING_APPROVAL` once `ValidationAgent` finishes (typically 5–30 s), with the line items, estimated total, and risk flags visible.
2. The reviewer reads the validation summary and clicks Approve. This POSTs to `/api/orders/{orderId}/approve`. The workflow resumes, `FulfillmentAgent` runs, and the tracking id appears with status `FULFILLED`.
3. The reviewer clicks Reject with a reason. This POSTs to `/api/orders/{orderId}/reject`. The order moves to terminal `REJECTED` and the reason is shown. The fulfillment step never runs.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| ValidationAgent | AutonomousAgent | Validates order and computes estimate; returns `OrderReview{lineItems, estimatedTotal, riskFlags}` | OrderWorkflow | OrderEntity |
| FulfillmentAgent | AutonomousAgent | Dispatches an approved order; returns `DispatchConfirmation{trackingId, dispatchedAt}` | OrderWorkflow | OrderEntity |
| OrderWorkflow | Workflow | Orchestrates validate → await approval → fulfill | OrderEndpoint | ValidationAgent, FulfillmentAgent, OrderEntity |
| OrderEntity | EventSourcedEntity | Holds the order state and lifecycle events | OrderWorkflow, OrderEndpoint | OrdersView |
| OrdersView | View | CQRS read model of all orders | OrderEntity | OrderEndpoint |
| OrderEndpoint | HttpEndpoint | REST + SSE + metadata surface | UI client | OrderWorkflow, OrderEntity, OrdersView |
| AppEndpoint | HttpEndpoint | Serves the static UI | Browser | static-resources |

## 5. Data model

`Order` (OrderEntity state and OrdersView row): `id` (String), `customerId` (`Optional<String>`), `status` (OrderStatus enum), and lifecycle fields all `Optional<T>`: `submittedAt`, `lineItemsSummary`, `estimatedTotal`, `riskFlags`, `reviewedAt`, `approvedAt`, `approvedBy`, `approverNote`, `rejectedAt`, `rejectedBy`, `rejectReason`, `fulfilledAt`, `trackingId`. Every nullable lifecycle field is `Optional<T>` (Lesson 6).

`OrderStatus` enum: `PENDING_APPROVAL`, `APPROVED`, `REJECTED`, `FULFILLED`.

Events: `OrderSubmitted`, `OrderApproved`, `OrderRejected`, `OrderFulfilled`.

Domain records: `OrderReview(String lineItems, String estimatedTotal, String riskFlags)`, `ApprovalDecision(String approvedBy, String note)`, `DispatchConfirmation(String trackingId, String dispatchedAt)`.

Full detail: `reference/data-model.md`.

## 6. API contract

Inline surface (schemas in `reference/api-contract.md`):

```
POST /api/orders                        -> { orderId }
POST /api/orders/{orderId}/approve      -> 200 | 404
POST /api/orders/{orderId}/reject       -> 200 | 404
GET  /api/orders                        -> { orders: [Order, ...] }
GET  /api/orders/{orderId}              -> Order
GET  /api/orders/sse                    -> Server-Sent Events of Order
GET  /api/metadata/eval-matrix          -> text/yaml
GET  /api/metadata/risk-survey          -> text/yaml
GET  /api/metadata/readme               -> text/markdown
GET  /                                  -> 302 /app/index.html
GET  /app/{*path}                       -> static-resources/{*path}
```

## 7. UI

Single self-contained `src/main/resources/static-resources/index.html` (no `ui/` folder, no npm build). Browser title: `<title>Akka Sample: HITL Order Processing</title>`. Five tabs: Overview / Architecture / Risk Survey / Eval Matrix / App UI. Tab switching matches by `data-tab` / `data-panel` attribute, never NodeList index, and removed tabs are deleted from the DOM, not hidden (Lesson 26). The Architecture tab renders the PLAN mermaid diagrams with the theme variables and CSS overrides from Lesson 24. The App UI tab submits an order, lists orders live via SSE, and shows Approve/Reject buttons on `PENDING_APPROVAL` orders that have a review summary. Full description: `reference/ui-mockup.md`.

## 8. Governance

Controls live in `eval-matrix.yaml`; the deployer survey in `risk-survey.yaml`. Mechanisms the generated system wires:

- **H1 — hitl · application.** `OrderWorkflow` pauses at the await-approval task; `/api/orders/{id}/approve` and `/api/orders/{id}/reject` resume it.
- **G1 — guardrail · before-tool-call.** A guardrail on `FulfillmentAgent` verifies `OrderEntity.status == APPROVED` before the simulated dispatch tool runs.
- **G2 — guardrail · before-agent-response.** A guardrail on `ValidationAgent` checks that the review contains at least a line-items summary and a total before the `OrderReview` is persisted for review.

## 9. Agent prompts

- `ValidationAgent` — validates the submitted order and produces a pricing estimate. See `prompts/validation-agent.md`.
- `FulfillmentAgent` — dispatches an approved order and returns a tracking id. See `prompts/fulfillment-agent.md`.

## 10. Acceptance

Inlined journeys (full set in `reference/user-journeys.md`):

1. **Submit an order.** POST an order; within ~30 s an order appears in `PENDING_APPROVAL` with a non-empty `lineItemsSummary` and `estimatedTotal`.
2. **Approve and fulfill.** Approve a `PENDING_APPROVAL` order; it reaches `FULFILLED` with a non-null `trackingId` within ~30 s.
3. **Reject an order.** Reject a `PENDING_APPROVAL` order with a reason; it moves to terminal `REJECTED` and the reason shows.
4. **Fulfillment guard.** The fulfillment step is never reached for an order that is not `APPROVED`.

---

## 11. Implementation directives

```
Create a sample named order-processing demonstrating the human-in-loop-gate ×
ops-automation cell. Runs out of the box (no external services). Maven group
io.akka.samples. Maven artifact human-in-loop-gate-ops-automation-order-processing-hitl.
Java package io.akka.samples.orderprocessing. Akka 3.6.0. HTTP port 9442.

Components to wire (exactly):
- 2 AutonomousAgents: ValidationAgent (validates the order and produces a pricing
  estimate, returns a typed OrderReview{lineItems, estimatedTotal, riskFlags}) and
  FulfillmentAgent (dispatches the order, returns a typed
  DispatchConfirmation{trackingId, dispatchedAt}). Each declares definition()
  returning an AgentDefinition with .instructions(...) loaded from prompts and
  .capability(TaskAcceptance.of(task).maxIterationsPerTask(3)). Both extend
  akka.javasdk.agent.autonomous.AutonomousAgent — never downgrade to Agent.
- 1 Workflow OrderWorkflow with three tasks: validateStep -> awaitApprovalStep
  -> fulfillStep. validateStep calls forAutonomousAgent(ValidationAgent.class,...)
  .runSingleTask(...) then forTask(taskId).result(...), writes recordReview on
  OrderEntity. awaitApprovalStep polls OrderEntity.getOrder; on PENDING_APPROVAL it
  self-schedules a 5-second resume timer; on APPROVED it transitions to fulfillStep;
  on REJECTED it ends. fulfillStep calls FulfillmentAgent and writes
  recordFulfillment. Override settings() with stepTimeout(60s) on validateStep and
  fulfillStep; WorkflowSettings is nested in Workflow (no import).
- 1 EventSourcedEntity OrderEntity holding an Order record with id, customerId
  (Optional<String>), OrderStatus enum
  {PENDING_APPROVAL,APPROVED,REJECTED,FULFILLED}, and Optional lifecycle fields
  (submittedAt, lineItemsSummary, estimatedTotal, riskFlags, reviewedAt, approvedAt,
  approvedBy, approverNote, rejectedAt, rejectedBy, rejectReason, fulfilledAt,
  trackingId). Events: OrderSubmitted, OrderApproved, OrderRejected, OrderFulfilled.
  Commands: recordReview, approve, reject, recordFulfillment, getOrder.
  emptyState() returns Order.initial("") with no commandContext() reference
  (Lesson 3).
- 1 View OrdersView with row type Order, table updater consuming OrderEntity events.
  ONE query: getAllOrders SELECT * AS orders FROM orders_view. No WHERE status filter
  (Akka cannot auto-index enum columns, Lesson 2) — filter client-side in callers.
- 2 HttpEndpoints: OrderEndpoint at /api with order submission (starts an
  OrderWorkflow with a fresh UUID), approve, reject, orders list (filter client-side
  from getAllOrders), single order, SSE stream, and three /api/metadata/* endpoints
  serving the YAML/MD files from src/main/resources/metadata/. AppEndpoint serving
  / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- OrderTasks.java declaring two Task<R> constants: VALIDATE (resultConformsTo
  OrderReview) and FULFILL (resultConformsTo DispatchConfirmation).
- OrderReview(String lineItems, String estimatedTotal, String riskFlags),
  ApprovalDecision(String approvedBy, String note),
  DispatchConfirmation(String trackingId, String dispatchedAt).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9442
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (H1, G1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data.types,
  capability.*, model.*, subjects.children; marking jurisdictions, declared
  frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: elevator pitch, component inventory, matrix cell,
  integration descriptor, how to run, the 5 tabs, an ASCII architecture diagram,
  project layout, API contract, license. NO governance-mechanisms section. NO
  configuration section.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN imports for
  markdown and YAML libs are acceptable. Five tabs: Overview, Architecture, Risk
  Survey, Eval Matrix, App UI. Match the governance.html visual style
  (dark/yellow/Instrument Sans/dot-grid). Include the Lesson 24 mermaid CSS
  overrides and theme variables. Tab switching matches by data-tab/data-panel
  attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent canned/random outputs
  (ValidationAgent -> OrderReview, FulfillmentAgent -> DispatchConfirmation; see
  src/main/resources/mock-responses/{validation-agent,fulfillment-agent}.json with
  4–6 entries each). Sets model-provider = mock.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed to
  the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Akka records only the
  REFERENCE — env-var name, file path, secrets URI — never the value.
- Bootstrap.java fails fast with a clear message naming the configured reference if
  it does not resolve at runtime; never echoes any captured key.

Mock LLM provider — per-agent mock-response shapes (option a):
- validation-agent.json: 4–6 entries, each { "lineItems": "...", "estimatedTotal":
  "...", "riskFlags": "..." }.
- fulfillment-agent.json: 4–6 entries, each { "trackingId": "TRK-<8 chars>",
  "dispatchedAt": "ISO-8601" }.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: stepTimeout(60s) on every agent-calling workflow step.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: each AutonomousAgent has its companion Task<R> constant in
  OrderTasks.java.
- Lesson 8: verify model names are current before locking application.conf.
- Lesson 9: the run command is "/akka:build", never "mvn akka:run".
- Lesson 10: port 9442 declared in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never
  T1/T2/T3/T4 or "deferred".
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: index.html includes mermaid CSS overrides and theme variables
  (state-label colour, edge-label overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: five-option key sourcing; never write a key value to disk.
- Lesson 26: tab switching matches by data-tab/data-panel attribute, never
  NodeList index; no hidden zombie panels in the DOM.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6.
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
