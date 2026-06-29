# SPEC — supply-chain-team

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Multi-Agent Supply Chain Optimizer.
**One-line pitch:** Submit a demand signal; a coordinator delegates inventory analysis and route planning to two specialist agents in parallel, then synthesises a single supply chain recommendation.

## 2. What this blueprint demonstrates

The **delegation-supervisor-workers** coordination pattern wired with Akka's first-party primitives: a Workflow fans work out to two AutonomousAgents in parallel, gathers their results, and asks a third AutonomousAgent to synthesise a unified supply chain recommendation. The blueprint also demonstrates an **output guardrail** that vets the recommendation before it is returned, and **eval-event** governance that samples the coordinator's synthesis decision for a quality score.

## 3. User-facing flows

The user opens the App UI tab and submits a demand signal (a product SKU or category with a quantity target) via the form.

1. The system creates a `SupplyOrder` record in `PENDING` and starts an `OptimizationWorkflow`.
2. The Coordinator decomposes the signal into two parallel work items: a stock-position query for the StockAnalyst and a routing brief for the LogisticsPlanner.
3. The workflow forks: both agents run concurrently. Each returns a typed payload.
4. The Coordinator merges the two payloads into a `SupplyRecommendation { summary, stockAssessment, routePlan, guardrailVerdict }`.
5. An output guardrail vets the recommendation; if it fails, the order moves to `BLOCKED`. Otherwise, the order moves to `RECOMMENDED`.
6. If either worker times out after 60 seconds, the workflow short-circuits: it asks the Coordinator to synthesise from whichever side returned, and the order enters `DEGRADED`.

A `DemandSimulator` (TimedAction) drips a canned demand signal every 60 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DemandCoordinator` | `AutonomousAgent` | Decomposes the demand signal into work items, synthesises the merged recommendation, runs the output guardrail. | `OptimizationWorkflow` | returns typed result to workflow |
| `StockAnalyst` | `AutonomousAgent` | Evaluates current inventory positions and stock-out risk for the requested SKUs. Seeded "inventory tool" returns canned stock levels. | `OptimizationWorkflow` | — |
| `LogisticsPlanner` | `AutonomousAgent` | Proposes replenishment routes and delivery schedules given lead times and carrier capacity. | `OptimizationWorkflow` | — |
| `OptimizationWorkflow` | `Workflow` | Coordinates the parallel fan-out, the synthesis, the guardrail. | `SupplyEndpoint`, `DemandSignalConsumer` | `SupplyOrderEntity` |
| `SupplyOrderEntity` | `EventSourcedEntity` | Holds the order's lifecycle (pending → analyzing → recommended / degraded / blocked). | `OptimizationWorkflow` | `SupplyOrderView` |
| `OrderQueueEntity` | `EventSourcedEntity` | Logs each submitted demand signal for replay and audit. | `SupplyEndpoint`, `DemandSimulator` | `DemandSignalConsumer` |
| `SupplyOrderView` | `View` | List-of-orders read model. | `SupplyOrderEntity` events | `SupplyEndpoint` |
| `DemandSignalConsumer` | `Consumer` | Listens to `OrderQueueEntity` events and starts one workflow per signal. | `OrderQueueEntity` events | `OptimizationWorkflow` |
| `DemandSimulator` | `TimedAction` | Drips a canned demand signal every 60 s. | scheduler | `OrderQueueEntity` |
| `FulfillmentEvalSampler` | `TimedAction` | Samples one completed recommendation every 5 minutes for eval scoring; emits a `RecommendationScored` event. | scheduler | `SupplyOrderEntity` |
| `SupplyEndpoint` | `HttpEndpoint` | `/api/supply/*` — submit, get, list, SSE. | — | `SupplyOrderView`, `OrderQueueEntity`, `SupplyOrderEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record DemandSignal(String sku, int targetQuantity, String requestedBy) {}

record StockPosition(String sku, int onHand, int inTransit, int reorderPoint,
                     boolean stockOutRisk, Instant assessedAt) {}

record StockAssessment(List<StockPosition> positions, String overallRisk,
                       Instant assessedAt) {}

record RouteSegment(String origin, String destination, int transitDays,
                    String carrier, double costUsd) {}

record RoutePlan(List<RouteSegment> segments, int totalTransitDays,
                 double totalCostUsd, Instant plannedAt) {}

record WorkOrder(String stockQuery, String routingBrief) {}

record SupplyRecommendation(String summary, StockAssessment stockAssessment,
                             RoutePlan routePlan, String guardrailVerdict,
                             Instant synthesisedAt) {}

record SupplyOrder(
    String orderId,
    String sku,
    int targetQuantity,
    OrderStatus status,
    Optional<StockAssessment> stockAssessment,
    Optional<RoutePlan> routePlan,
    Optional<SupplyRecommendation> recommendation,
    Optional<String> failureReason,
    Optional<Integer> evalScore,
    Optional<String> evalRationale,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum OrderStatus { PENDING, ANALYZING, RECOMMENDED, DEGRADED, BLOCKED }
```

### Events (on `SupplyOrderEntity`)

`OrderCreated`, `StockAssessmentAttached`, `RoutePlanAttached`, `OrderRecommended`, `OrderDegraded`, `OrderBlocked`, `RecommendationScored`.

### Events (on `OrderQueueEntity`)

`DemandSignalSubmitted { orderId, sku, targetQuantity, requestedBy, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/supply` — body `{ sku, targetQuantity }` → `{ orderId }`. Starts a workflow.
- `GET /api/supply` — list all orders. Optional `?status=PENDING|ANALYZING|RECOMMENDED|DEGRADED|BLOCKED`.
- `GET /api/supply/{id}` — one order.
- `GET /api/supply/sse` — server-sent events stream of every order change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Multi-Agent Supply Chain Optimizer"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a demand signal (SKU + quantity), live list of orders with status pills, expand-row to see stock assessment + route plan + synthesised recommendation + eval score.

Browser title: `<title>Akka Sample: Multi-Agent Supply Chain Optimizer</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — output guardrail** (`before-agent-response` on `DemandCoordinator`): vets the supply recommendation for infeasible routes, negative stock quantities, and policy violations. Blocking. Failure → `BLOCKED`.
- **E1 — eval-event sampler** (`on-decision-eval`): `FulfillmentEvalSampler` (TimedAction) picks one completed recommendation every 5 minutes and emits a `RecommendationScored` event with a 1–5 score and a short rationale.

## 9. Agent prompts

- `DemandCoordinator` → `prompts/demand-coordinator.md`. Decomposes the demand signal into parallel work items; later synthesises results into the final recommendation.
- `StockAnalyst` → `prompts/stock-analyst.md`. Evaluates inventory positions; returns `StockAssessment`.
- `LogisticsPlanner` → `prompts/logistics-planner.md`. Proposes replenishment routes; returns `RoutePlan`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit a demand signal; order progresses PENDING → ANALYZING → RECOMMENDED within 60 s; UI reflects each transition via SSE.
2. **J2** — Inject a worker timeout (set `StockAnalyst` timeout to 1 s); order enters DEGRADED with whichever partial output came back.
3. **J3** — Inject a guardrail failure (Coordinator returns an infeasible route); order enters BLOCKED.
4. **J4** — Wait after a successful recommendation; the order's row in the UI shows an eval score.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named supply-chain-team demonstrating the
delegation-supervisor-workers × ops-automation cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
delegation-supervisor-workers-ops-automation-supply-chain-team.
Java package io.akka.samples.supplychain. Akka 3.6.0. HTTP port 9488.

Components to wire (exactly):
- 3 AutonomousAgents:
  * DemandCoordinator — definition() with capability(TaskAcceptance.of(DECOMPOSE).maxIterationsPerTask(2))
    AND capability(TaskAcceptance.of(SYNTHESISE).maxIterationsPerTask(3)). System prompt loaded
    from prompts/demand-coordinator.md. Returns WorkOrder{stockQuery, routingBrief} for DECOMPOSE
    and SupplyRecommendation{summary, stockAssessment, routePlan, guardrailVerdict, synthesisedAt}
    for SYNTHESISE.
  * StockAnalyst — capability(TaskAcceptance.of(ASSESS).maxIterationsPerTask(3)). System prompt
    from prompts/stock-analyst.md. Returns StockAssessment{positions: List<StockPosition{sku,
    onHand, inTransit, reorderPoint, stockOutRisk, assessedAt}>, overallRisk, assessedAt}.
  * LogisticsPlanner — capability(TaskAcceptance.of(PLAN).maxIterationsPerTask(2)). System prompt
    from prompts/logistics-planner.md. Returns RoutePlan{segments: List<RouteSegment{origin,
    destination, transitDays, carrier, costUsd}>, totalTransitDays, totalCostUsd, plannedAt}.

- 1 Workflow OptimizationWorkflow with steps:
  decomposeStep -> [parallel] assessStep, planStep -> joinStep -> synthesiseStep -> guardrailStep -> emitStep.
  decomposeStep calls forAutonomousAgent(DemandCoordinator.class, DECOMPOSE).
  assessStep and planStep run in parallel (CompletionStage zip); each governed by
  WorkflowSettings.builder().stepTimeout(OptimizationWorkflow::assessStep, ofSeconds(60)) and
  stepTimeout(OptimizationWorkflow::planStep, ofSeconds(60)). On either timeout, transition to a
  degradeStep that calls synthesiseStep with whichever side returned, then ends with OrderDegraded.
  synthesiseStep calls forAutonomousAgent(DemandCoordinator.class, SYNTHESISE) with the merged
  inputs; give synthesiseStep a 90s stepTimeout. guardrailStep runs the deterministic vetter +
  LLM judge on the synthesised recommendation; on failure, end with OrderBlocked.
  WorkflowSettings is nested inside Workflow — no import.

- 1 EventSourcedEntity SupplyOrderEntity holding state SupplyOrder{orderId, sku,
  targetQuantity, OrderStatus, Optional<StockAssessment> stockAssessment,
  Optional<RoutePlan> routePlan, Optional<SupplyRecommendation> recommendation,
  Optional<String> failureReason, Optional<Integer> evalScore,
  Optional<String> evalRationale, Instant createdAt, Optional<Instant> finishedAt}.
  OrderStatus enum: PENDING, ANALYZING, RECOMMENDED, DEGRADED, BLOCKED.
  Events: OrderCreated, StockAssessmentAttached, RoutePlanAttached, OrderRecommended,
  OrderDegraded, OrderBlocked, RecommendationScored. Commands: createOrder,
  attachStockAssessment, attachRoutePlan, recommend, degrade, block, recordEval, getOrder.
  emptyState() returns SupplyOrder.initial("", null) with no commandContext() reference.

- 1 EventSourcedEntity OrderQueueEntity with command enqueueSignal(sku, targetQuantity, requestedBy)
  emitting DemandSignalSubmitted{orderId, sku, targetQuantity, requestedBy, submittedAt}.

- 1 View SupplyOrderView with row type SupplyOrderRow (mirrors SupplyOrder minus heavy
  nested payloads; every nullable field is Optional<T>). Table updater consumes
  SupplyOrderEntity events. ONE query getAllOrders SELECT * AS orders FROM supply_order_view.
  No WHERE status filter (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer DemandSignalConsumer subscribed to OrderQueueEntity events; on DemandSignalSubmitted
  starts an OptimizationWorkflow with the orderId as the workflow id.

- 2 TimedActions:
  * DemandSimulator — every 60s, reads next line from
    src/main/resources/sample-events/demand-signals.jsonl and calls OrderQueueEntity.enqueueSignal.
  * FulfillmentEvalSampler — every 5 minutes, queries SupplyOrderView.getAllOrders, picks the oldest
    RECOMMENDED order without an evalScore, runs a 1-5 rubric judge over the recommendation
    content, then calls SupplyOrderEntity.recordEval(score, rationale).

- 2 HttpEndpoints:
  * SupplyEndpoint at /api with POST /supply, GET /supply, GET /supply/{id},
    GET /supply/sse, and the /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- SupplyTasks.java declaring four Task<R> constants: DECOMPOSE (WorkOrder), ASSESS
  (StockAssessment), PLAN (RoutePlan), SYNTHESISE (SupplyRecommendation).
- Domain records WorkOrder, DemandSignal, StockPosition, StockAssessment, RouteSegment,
  RoutePlan, SupplyRecommendation.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9488 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/demand-signals.jsonl with 8 canned demand signal lines
  covering SKUs across electronics, apparel, and food categories.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1 output guardrail
  before-agent-response, E1 eval-event on-decision-eval) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose.primary_function = supply-chain-
  optimization, decisions.authority_level = recommend-only, data.data_classes.pii = false,
  capabilities.* = false; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/demand-coordinator.md, prompts/stock-analyst.md, prompts/logistics-planner.md loaded
  at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Multi-Agent Supply Chain Optimizer",
  one-line pitch, prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file (no ui/
  folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams + click-to-expand
  component table with syntax-highlighted Java snippets), Risk Survey (7 sub-tabs from
  governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column ID/Control/Mechanism/
  Implementation/Source table with click-to-expand rows), App UI (form + live list with status
  pills). Browser title exactly: <title>Akka Sample: Multi-Agent Supply Chain Optimizer</title>.
  No subtitle on the Overview tab.

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
  src/main/resources/mock-responses/<agent-name>.json (demand-coordinator.json,
  stock-analyst.json, logistics-planner.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    demand-coordinator.json — list of either WorkOrder or SupplyRecommendation objects.
      4-6 WorkOrder entries (stockQuery + routingBrief pairs targeting realistic SKUs) and
      4-6 SupplyRecommendation entries (each with an 80-150 word summary, a 2-4 position
      stock assessment, a 2-4 segment route plan, guardrailVerdict = "ok").
    stock-analyst.json — 4-6 StockAssessment entries, each with 2-4 StockPosition records
      whose sku values are realistic product codes (e.g., "SKU-ELEC-9821", "SKU-APL-0042"),
      onHand and inTransit are non-negative integers, overallRisk is "low", "medium", or
      "high".
    logistics-planner.json — 4-6 RoutePlan entries, each with 2-4 RouteSegment records
      whose carrier values are realistic (e.g., "FedEx Freight", "UPS Ground", "USPS Priority"),
      costUsd is a positive number, totalTransitDays is between 1 and 14.
- A MockModelProvider.seedFor(orderId) helper makes the selection
  deterministic per order id so the same order produces the same output
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls an agent gets an explicit stepTimeout (60s workers,
  90s synthesis); default 5s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion SupplyTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o, gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9488 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible, transitionLabelColor
  #cccccc). Without these, state names render black-on-black and arrow labels clip.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
- Parallel workflow steps use CompletionStage zip, NOT sequential calls.
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

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
