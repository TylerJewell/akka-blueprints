# Sample Specification — `negotiation`

This document is the **natural-language input for `/akka:specify`**. The user runs `/akka:specify @SPEC.md` from inside this folder; the `@SPEC.md` token inlines the whole file as the skill's argument. Sections 1–11 together are the spec; Section 12 tells Claude how to continue after scaffolding.

---

## 1. System name + one-line pitch

**Negotiation Facilitator.** A user submits a negotiation request — an item, a buyer budget, and a seller floor. A Facilitator runs up to ten rounds of offers and counteroffers between a Buyer and a Seller until terms converge, then returns the outcome (a deal or no deal) and the final offer.

## 2. What this blueprint demonstrates

The **moderation-turn-taking** coordination pattern: a moderator (the Facilitator) holds the turn order between two opposed parties (Buyer and Seller), gives each party exactly one turn per round, and after both have moved decides whether the round produced convergence, a breakdown, or another round. Up to ten rounds run before the moderator must conclude. The **governance pattern** pairs an output guardrail on every agent response — so no agent can commit terms outside its declared constraints and the Facilitator cannot fabricate a final price outside the budget-to-floor band — with an automated evaluator that fires on the stop decision and scores the concluded outcome. Every component is a first-party Akka primitive in one buildable folder; the market, the request stream, and the operator halt switch are all modeled inside the same service.

## 3. User-facing flows

1. **Submit a negotiation.** The user fills in an item, a buyer budget, and a seller floor in the App UI tab and clicks Start. The service returns a `negotiationId` and the row appears in `NEGOTIATING` state.
2. **Watch the rounds.** The live list streams each offer as it lands — round number, party (Buyer or Seller), price, and one-line rationale — so the user sees the gap narrow turn by turn.
3. **See the outcome.** When the Facilitator decides, the row moves to `CONCLUDED` with an outcome of `CONVERGED` (a final price and final terms) or `NO_DEAL`, and an evaluator score appears beside it.
4. **Let it run on its own.** Without any input, the request simulator drips a canned scenario every thirty seconds, so negotiations keep arriving and concluding.
5. **Halt the system.** An operator can flip a halt switch; queued requests stop starting new negotiations until the operator resumes.

These flows are the acceptance journeys in [`reference/user-journeys.md`](reference/user-journeys.md).

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BuyerAgent` | AutonomousAgent | Produces the buyer's offer or counteroffer for the current round; returns a typed `Offer` | `NegotiationWorkflow` | `NegotiationEntity` (via workflow) |
| `SellerAgent` | AutonomousAgent | Produces the seller's counteroffer for the current round; returns a typed `Offer` | `NegotiationWorkflow` | `NegotiationEntity` (via workflow) |
| `FacilitatorAgent` | AutonomousAgent | Reads both latest offers each round and returns a typed `FacilitatorDecision` of continue / converged / no-deal, with final terms on convergence | `NegotiationWorkflow` | `NegotiationEntity` (via workflow) |
| `NegotiationTasks` | task definitions | Declares the `Task<Offer>` and `Task<FacilitatorDecision>` constants the three agents accept | — | the three agents |
| `NegotiationWorkflow` | Workflow | The Facilitator's turn-taking loop: `openStep` → `buyerTurnStep` → `sellerTurnStep` → `facilitateStep`, repeating up to ten rounds, then `concludeStep` | `NegotiationRequestConsumer` | the three agents, `NegotiationEntity` |
| `NegotiationEntity` | EventSourcedEntity | Durable per-negotiation state: every offer, the status, the concluded outcome and final offer, the evaluator score | `NegotiationWorkflow`, `OutcomeEvaluator`, `StalledNegotiationMonitor` | `NegotiationsView` |
| `InboundRequestQueue` | EventSourcedEntity | Records each incoming negotiation request as an event | `RequestSimulator`, `NegotiationEndpoint` | `NegotiationRequestConsumer` |
| `SystemControl` | KeyValueEntity | Holds the operator halt flag | `NegotiationEndpoint` | `NegotiationRequestConsumer` |
| `NegotiationsView` | View | Row type `Negotiation`; one query returning all negotiations for the UI list and SSE stream | `NegotiationEntity` events | `NegotiationEndpoint`, `StalledNegotiationMonitor` |
| `NegotiationRequestConsumer` | Consumer | On each queued request, checks the halt flag and starts a `NegotiationWorkflow` with a fresh id | `InboundRequestQueue` events | `NegotiationWorkflow` |
| `OutcomeEvaluator` | Consumer | On `NegotiationConcluded`, scores the outcome and records it on the entity — this is the on-decision evaluator | `NegotiationEntity` events | `NegotiationEntity` |
| `RequestSimulator` | TimedAction | Every thirty seconds, drips the next canned scenario from a JSONL file into the queue | scheduled tick | `InboundRequestQueue` |
| `StalledNegotiationMonitor` | TimedAction | Every thirty seconds, marks negotiations still running after two minutes as `ESCALATED` | scheduled tick, `NegotiationsView` | `NegotiationEntity` |
| `NegotiationEndpoint` | HttpEndpoint | The `/api/*` surface: start, list, single, SSE, halt/resume, and the three metadata endpoints | browser, simulator | `InboundRequestQueue`, `NegotiationsView`, `SystemControl` |
| `AppEndpoint` | HttpEndpoint | Serves `/` (redirect) and `/app/*` (static UI) | browser | static-resources |
| `Bootstrap` | service-setup | On startup, schedules the two TimedActions | runtime | the TimedActions |

Names are authoritative — `/akka:implement` uses them verbatim.

## 5. Data model

Full field-by-field detail is in [`reference/data-model.md`](reference/data-model.md). Authoritative summary:

`Negotiation` is the event-sourced state **and** the View row. Every field that is null until a later event fires is `Optional<T>` (Lesson 6).

```java
public record Negotiation(
  String id,
  Optional<String>  item,
  Optional<Double>  buyerBudget,
  Optional<Double>  sellerFloor,
  NegotiationStatus status,
  int               currentRound,
  List<OfferLine>   offers,            // append-only history, never null (empty list initially)
  Optional<Double>  latestBuyerPrice,
  Optional<Double>  latestSellerPrice,
  Optional<String>  outcome,           // "CONVERGED" | "NO_DEAL" once concluded
  Optional<Double>  finalPrice,
  Optional<String>  finalTerms,
  Optional<Instant> startedAt,
  Optional<Instant> concludedAt,
  Optional<Instant> escalatedAt,
  Optional<Double>  outcomeScore,      // set by OutcomeEvaluator
  Optional<String>  outcomeNotes
) {
  public static Negotiation initial(String id) { /* status CREATED, round 0, empty offers, all Optional.empty() */ }
  public Negotiation applyEvent(NegotiationEvent e) { /* per-variant switch */ }
}
```

`OfferLine(int round, String party, double price, String terms, String rationale, boolean accept, Instant at)` — `party` is `"BUYER"` or `"SELLER"`.

`NegotiationStatus` enum: `CREATED · NEGOTIATING · CONCLUDED · ESCALATED`. The deal-versus-no-deal split is carried by the `outcome` string, not the enum, so no view query filters on the enum.

`NegotiationEvent` (sealed, six variants): `NegotiationStarted`, `OfferRecorded`, `RoundAdvanced`, `NegotiationConcluded`, `NegotiationEscalated`, `OutcomeEvaluated`.

Agent result records: `Offer(double price, String terms, String rationale, boolean accept)` and `FacilitatorDecision(String verdict, double finalPrice, String finalTerms, String reasoning)` where `verdict` is `CONTINUE` / `CONVERGED` / `NO_DEAL`.

## 6. API contract

Every endpoint, payload schema, and the SSE event format are in [`reference/api-contract.md`](reference/api-contract.md). Top-level surface (all JSON, ACL open for local-dev):

```
POST /api/negotiations                 -> { negotiationId }
GET  /api/negotiations ?status=...     -> { negotiations: [Negotiation, ...] }   (status filtered client-side)
GET  /api/negotiations/{id}            -> Negotiation
GET  /api/negotiations/sse             -> Server-Sent Events of Negotiation
POST /api/system/halt                  -> { halted: true }
POST /api/system/resume                -> { halted: false }
GET  /api/system/status                -> { halted: bool }

GET  /api/metadata/eval-matrix         -> text/yaml
GET  /api/metadata/risk-survey         -> text/yaml
GET  /api/metadata/readme              -> text/markdown

GET  /                                 -> 302 /app/index.html
GET  /app/{*path}                      -> static-resources/{*path}
```

## 7. UI

The UI is a single self-contained `src/main/resources/static-resources/index.html` (Lesson 17): inline CSS and JS, runtime CDN imports for markdown and YAML rendering are acceptable, no `ui/` folder and no build step. Browser title: `<title>Akka Sample: Negotiation Facilitator</title>`. Full description in [`reference/ui-mockup.md`](reference/ui-mockup.md). Five tabs:

1. **Overview** — eyebrow "Overview" + headline (sample type), no subtitle, then the four cards Try it / How it works / Components / API contract.
2. **Architecture** — the four mermaid diagrams (component graph, sequence, state machine, entity model) with the Lesson 24 CSS overrides, plus a click-to-expand component table.
3. **Risk Survey** — renders `/api/metadata/risk-survey` in the `matrix-card` / `matrix-row` style; values matching `TO_BE_COMPLETED_BY_DEPLOYER` render muted and italic.
4. **Eval Matrix** — renders `/api/metadata/eval-matrix` in the same style; the label column carries the control id plus a colored mechanism pill.
5. **App UI** — the live interaction: a Start form (item, buyer budget, seller floor), the SSE-streamed list of negotiations with their per-round offers expanding inline, the concluded outcome and evaluator score, and the operator halt / resume control.

Tab switching matches by `data-tab` → `data-panel` attribute, never by NodeList index, and removed tabs are deleted from the DOM rather than hidden (Lesson 26). Mermaid state-label color and edge-label overflow follow Lesson 24.

## 8. Governance

The controls the generated system must wire are in [`eval-matrix.yaml`](eval-matrix.yaml); the deployer risk posture is in [`risk-survey.yaml`](risk-survey.yaml). One sentence per mechanism:

- **G1 — output guardrail (`guardrail` · before-agent-response).** A before-agent-response guardrail on `BuyerAgent`, `SellerAgent`, and `FacilitatorAgent` rejects any offer outside the agent's declared band (buyer never above budget, seller never below floor) and rejects any `FacilitatorDecision` whose final price falls outside the seller-floor-to-buyer-budget band, because final negotiated terms carry commercial implications.
- **E1 — outcome evaluator (`eval-event` · on-decision-eval).** `OutcomeEvaluator` subscribes to `NegotiationConcluded` and scores the stop decision (surplus split between the parties, rounds used, deal-or-no-deal), recording the score on the entity for the UI.
- **A1 — test gate (`ci-gate` · test-gate).** The turn-taking loop and the convergence math must pass an integration test before any deploy.
- **HT1 — operator halt (`halt` · operator-regulator-stop).** `SystemControl` holds a halt flag the endpoint and the request consumer check before starting new negotiations.

G1 and E1 are the pattern-specific controls drawn from the corpus entry; A1 and HT1 are the standard operational controls every deployable governed service carries.

## 9. Agent prompts

One file per agent under `prompts/`, pasted verbatim into the agent's instructions:

- [`prompts/buyer-agent.md`](prompts/buyer-agent.md) — the buyer's negotiating stance: open below budget, concede slowly, never exceed the budget.
- [`prompts/seller-agent.md`](prompts/seller-agent.md) — the seller's negotiating stance: open above the floor, concede slowly, never drop below the floor.
- [`prompts/facilitator-agent.md`](prompts/facilitator-agent.md) — the moderator's adjudication rule: declare converged when the gap is within tolerance, no-deal when the parties diverge or the round cap is reached, otherwise continue.

## 10. Acceptance

The full journeys are in [`reference/user-journeys.md`](reference/user-journeys.md). The system generated correctly when these pass:

1. **Start and observe.** `POST /api/negotiations` with `{ item, buyerBudget, sellerFloor }` where budget ≥ floor returns a `negotiationId`; within seconds the negotiation is `NEGOTIATING` and the Buyer's round-1 offer is present in `offers`.
2. **Convergence to a deal.** When the buyer budget comfortably exceeds the seller floor, the negotiation reaches `CONCLUDED` with `outcome = CONVERGED`, a `finalPrice` between the floor and the budget, and non-empty `finalTerms`, at or before round ten.
3. **No deal.** When the buyer budget is below the seller floor, the negotiation reaches `CONCLUDED` with `outcome = NO_DEAL` no later than round ten, and no `finalPrice` is set.
4. **Outcome scored.** Every `CONCLUDED` negotiation gets a non-empty `outcomeScore` from the evaluator within seconds of concluding.

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named negotiation demonstrating the
moderation-turn-taking x general cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact negotiation. Java
package io.akka.samples.negotiation. Akka 3.6.0. HTTP port 9375.

Components to wire (exactly):
- 3 AutonomousAgents:
  - BuyerAgent: given item, buyer budget, and the seller's last price,
    returns a typed Offer{price, terms, rationale, accept}. definition()
    declares capability(TaskAcceptance.of(BUYER_OFFER).maxIterationsPerTask(3)).
  - SellerAgent: given item, seller floor, and the buyer's last price,
    returns a typed Offer{price, terms, rationale, accept}. Same capability
    pattern with SELLER_OFFER.
  - FacilitatorAgent: given both latest offers, the round number, and the
    convergence tolerance, returns a typed FacilitatorDecision{verdict,
    finalPrice, finalTerms, reasoning}. Capability over FACILITATE.
  Each agent class extends akka.javasdk.agent.autonomous.AutonomousAgent and
  has a single definition() method — never silently downgraded to Agent.
- NegotiationTasks.java declaring three Task constants: BUYER_OFFER
  (resultConformsTo Offer.class), SELLER_OFFER (Offer.class), FACILITATE
  (FacilitatorDecision.class), each Task.name(...).description(...)
  .resultConformsTo(...).
- 1 Workflow NegotiationWorkflow with steps openStep -> buyerTurnStep ->
  sellerTurnStep -> facilitateStep -> (loop or concludeStep). openStep writes
  NegotiationStarted and goes to buyerTurnStep. buyerTurnStep calls
  forAutonomousAgent(BuyerAgent.class, "buyer-"+id).runSingleTask(BUYER_OFFER
  .instructions(...)) then forTask(taskId).result(BUYER_OFFER); records the
  offer via NegotiationEntity.recordOffer; transitions to sellerTurnStep.
  sellerTurnStep is symmetric with SellerAgent, then transitions to
  facilitateStep. facilitateStep calls FacilitatorAgent; on verdict CONVERGED
  transitions to concludeStep(CONVERGED); on NO_DEAL transitions to
  concludeStep(NO_DEAL); on CONTINUE increments the round via
  NegotiationEntity.advanceRound and, if the new round exceeds 10, transitions
  to concludeStep(NO_DEAL), else back to buyerTurnStep. concludeStep writes
  NegotiationConcluded with outcome, finalPrice, finalTerms, roundsUsed and
  ends. Override settings() with stepTimeout(60s) on buyerTurnStep,
  sellerTurnStep, facilitateStep, and concludeStep, and a
  defaultStepRecovery(maxRetries(2).failoverTo(error)).
- 1 EventSourcedEntity NegotiationEntity holding the Negotiation record (id,
  Optional item, Optional buyerBudget, Optional sellerFloor, NegotiationStatus
  enum, int currentRound, List<OfferLine> offers, Optional latestBuyerPrice,
  Optional latestSellerPrice, Optional outcome, Optional finalPrice, Optional
  finalTerms, Optional startedAt, Optional concludedAt, Optional escalatedAt,
  Optional outcomeScore, Optional outcomeNotes). Events: NegotiationStarted,
  OfferRecorded, RoundAdvanced, NegotiationConcluded, NegotiationEscalated,
  OutcomeEvaluated. Commands: start, recordOffer, advanceRound, conclude,
  markEscalated, recordEvaluation, getNegotiation. emptyState() returns
  Negotiation.initial("") with no commandContext() reference (Lesson 3).
- 1 EventSourcedEntity InboundRequestQueue, single instance "default", one
  command enqueueRequest(item, buyerBudget, sellerFloor) emitting
  NegotiationRequestQueued.
- 1 KeyValueEntity SystemControl, single instance "default", holding a
  boolean halted; commands halt, resume, isHalted.
- 1 View NegotiationsView with row type Negotiation, table updater consuming
  NegotiationEntity events. ONE query: getAllNegotiations SELECT * AS
  negotiations FROM negotiations_view. NO WHERE status filter (Akka cannot
  auto-index enum columns, Lesson 2) — callers filter client-side. Provide a
  streamAllNegotiations variant for the SSE endpoint.
- 1 Consumer NegotiationRequestConsumer subscribed to InboundRequestQueue
  events; on each event, reads SystemControl.isHalted; if not halted, starts a
  NegotiationWorkflow with a fresh UUID.
- 1 Consumer OutcomeEvaluator subscribed to NegotiationEntity events; on
  NegotiationConcluded, computes an outcome score (surplus split, rounds used,
  deal flag) and calls NegotiationEntity.recordEvaluation. Ignore other event
  variants.
- 2 TimedActions: RequestSimulator (every 30s, reads the next line from
  src/main/resources/sample-events/negotiation-requests.jsonl and calls
  InboundRequestQueue.enqueueRequest); StalledNegotiationMonitor (every 30s,
  queries NegotiationsView.getAllNegotiations, filters status NEGOTIATING with
  startedAt older than 2 minutes, calls NegotiationEntity.markEscalated).
- 2 HttpEndpoints: NegotiationEndpoint at /api with negotiations (POST start,
  GET list filtered client-side from getAllNegotiations, GET single, GET sse),
  system halt/resume/status backed by SystemControl, and three /api/metadata/*
  endpoints serving the YAML/MD files from src/main/resources/metadata/.
  AppEndpoint serving / -> 302 /app/index.html and /app/* ->
  static-resources/*.
- 1 service-setup Bootstrap scheduling RequestSimulator and
  StalledNegotiationMonitor on startup; fails fast (Lesson 25 step 4) with a
  clear message naming the configured key reference if the model provider key
  does not resolve, never echoing key material.

Companion files:
- Offer(double price, String terms, String rationale, boolean accept),
  FacilitatorDecision(String verdict, double finalPrice, String finalTerms,
  String reasoning), OfferLine(int round, String party, double price, String
  terms, String rationale, boolean accept, Instant at).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port =
  9375 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Verify model names are current before locking
  (Lesson 8).
- src/main/resources/sample-events/negotiation-requests.jsonl with 8 canned
  scenarios; mix budget>=floor (deal) and budget<floor (no deal) cases.
- src/main/resources/metadata/{eval-matrix.yaml, risk-survey.yaml, README.md}
  (copies of the project-root files so the endpoint serves them from classpath).
- eval-matrix.yaml at the project root with controls G1, E1, A1, HT1 and a
  matching simplified_view. No regulation_anchors.
- risk-survey.yaml at the project root pre-filling sector, decisions, data
  types, capability.*, model.*, subjects.children; marking jurisdictions,
  declared_frameworks, organization fields, incidents, and data.residency as
  TO_BE_COMPLETED_BY_DEPLOYER.
- README.md at the project root: pitch, component inventory, matrix cell,
  integration descriptor, how to run, the five tabs, an ASCII architecture
  diagram, project layout, API contract, license. NO governance-mechanisms
  section, NO configuration section (Lesson 20).
- src/main/resources/static-resources/index.html — one self-contained HTML
  file (Lesson 17), inline CSS + JS, runtime CDN imports for markdown and YAML
  acceptable. Five tabs (Overview, Architecture, Risk Survey, Eval Matrix, App
  UI). Match the governance.html visual style (dark / yellow accent /
  Instrument Sans / dot-grid). Include the Lesson 24 mermaid CSS overrides and
  theme variables; switch tabs by data-tab / data-panel attribute (Lesson 26).

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, inspect the environment for ANTHROPIC_API_KEY /
  OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set, default
  application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
  (a) Mock LLM — generate a MockModelProvider with per-agent schema-valid
  outputs (BuyerAgent -> Offer, SellerAgent -> Offer, FacilitatorAgent ->
  FacilitatorDecision); see src/main/resources/mock-responses/{buyer-agent,
  seller-agent,facilitator-agent}.json each with 4-6 entries. Sets
  model-provider = mock. The mock Buyer and Seller must converge within ten
  rounds on the deal scenarios and stay apart on the no-deal scenarios so the
  journeys still pass deterministically.
  (b) Name an existing env var — record the NAME in application.conf.
  (c) Point to an existing env file path — record the PATH in .akka-build.yaml;
  /akka:build sources the file before spawning the JVM.
  (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
  (e) Type once in this session — value lives in Claude session memory; passed
  to the JVM via the MCP tool's environment parameter; gone at session end.
- NEVER write the key value to any file Akka creates. Record only the
  REFERENCE — env-var name, file path, secrets URI — never the value.

Mock LLM provider per-agent output schemas (option (a)):
- buyer-agent.json: entries shaped Offer{price below buyerBudget and rising
  toward it across calls, terms, rationale, accept true only when the gap to
  the seller is within tolerance}.
- seller-agent.json: entries shaped Offer{price above sellerFloor and falling
  toward it, terms, rationale, accept true only when the gap is within
  tolerance}.
- facilitator-agent.json: entries shaped FacilitatorDecision{verdict CONTINUE
  while the gap exceeds tolerance and round <= 10, CONVERGED with finalPrice =
  midpoint of the two latest prices when within tolerance, NO_DEAL when round
  exceeds 10 or budget < floor, reasoning}.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Lesson 1: AutonomousAgent is never silently downgraded to Agent; the
  extends clause matches this spec verbatim.
- Lesson 4: every workflow step that calls an agent overrides stepTimeout to
  60s; the 5s default would time out on LLM calls.
- Lesson 6: every nullable lifecycle field on the Negotiation row record is
  Optional<T>; the offers list is never null (empty list initially).
- Lesson 7: AutonomousAgent requires the companion NegotiationTasks.java; the
  agent classes reference its Task constants.
- Lesson 8: verify model names against the provider's current lineup before
  writing model-name.
- Lesson 9: the run command is "/akka:build" everywhere user-facing, never
  "mvn akka:run".
- Lesson 10: port 9375 is declared explicitly in application.conf.
- Lesson 11: no source.platform or any corpus-internal provenance appears in
  any user-facing surface.
- Lesson 12: the UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration is described as "Runs out of the box", never a tier
  code, never "deferred".
- Lesson 23: no competitor brand names anywhere.
- Lesson 24: the index.html includes the mermaid state-label color overrides,
  edge-label foreignObject overflow:visible, and transitionLabelColor #cccccc.
- Lesson 25: the five-option key sourcing above; never a key value on disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never
  by NodeList index; removed tabs are deleted from the DOM, not display:none.
- Overview tab Try-it card is just "/akka:build" — no env-var export block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Continue through the rest of the pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 and 6 and [`PLAN.md`](PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL (`http://localhost:9375`) and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step runs longer than thirty seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — an unresolved model-provider key reference (offer the five key-sourcing options from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
