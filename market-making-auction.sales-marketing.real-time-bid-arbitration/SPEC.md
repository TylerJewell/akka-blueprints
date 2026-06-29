# SPEC — real-time-bid-arbitration

The natural-language brief `/akka:specify` reads to generate this system. Detail wins. Sections 1–11 together are the input to `/akka:specify @SPEC.md`.

---

## 1. System name + pitch

**System name:** Real-Time Bid Arbitration Agent.
**One-line pitch:** Submit an auction impression; the agent decides whether to bid and at what price, enforcing a per-window spend cap that halts further bids when exhausted.

## 2. What this blueprint demonstrates

The **market-making-auction** coordination pattern wired with Akka's first-party primitives: a Workflow drives each auction slot through evaluation and result recording, an AutonomousAgent decides bid price against campaign constraints, and a per-window budget guard halts bidding when the cap is reached. The blueprint also demonstrates a **periodic eval** that computes win-rate and ROAS against the campaign budget envelope, and a **HITL escalation** that routes anomalous bid-burst slots to an ops queue for human review.

## 3. User-facing flows

The user opens the App UI tab, selects a campaign, and watches the live bid ledger. The `AuctionSimulator` drips one sample impression every 30 seconds so the ledger is non-empty on first load.

1. An auction impression arrives (from the UI form or the simulator). The system creates an `AuctionSlot` in `RECEIVED` state and starts a `BidWorkflow`.
2. The workflow checks the campaign's current window spend. If the cap is already breached, the slot moves to `HALTED` immediately.
3. Otherwise, `BidArbitrationAgent` evaluates the impression context and returns a `BidDecision { shouldBid, bidPrice, reasoning }`.
4. If `shouldBid` is false, the slot moves to `SKIPPED`.
5. If `shouldBid` is true, the workflow places the bid against the simulated exchange and waits for a `BidResult`. The slot moves to `BID_PLACED`.
6. When the result arrives, the slot moves to `WON` or `LOST`. A `BidCharged` event on `CampaignBudgetEntity` updates the window spend; if spend now exceeds the cap, `CapBreached` is emitted and future slots in this window are halted.
7. A burst detector inside `BidWorkflow` counts bid placements in a rolling 60-second window. If the count exceeds the configured threshold, the slot moves to `ESCALATED` and a message is written to the ops queue.

`PerformanceSampler` (TimedAction) runs every 5 minutes, reads the `BidLedgerView`, computes win-rate and ROAS for the window, and writes a `PerformanceEvalRecorded` event to the relevant `AuctionSlotEntity`.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BidArbitrationAgent` | `AutonomousAgent` | Evaluates auction impression context against campaign goals; returns `BidDecision`. | `BidWorkflow` | returns typed result to workflow |
| `BidWorkflow` | `Workflow` | Drives per-slot lifecycle: budget check → evaluate → place bid → record result → charge budget. | `BidEndpoint`, `AuctionRequestConsumer` | `AuctionSlotEntity`, `CampaignBudgetEntity` |
| `AuctionSlotEntity` | `EventSourcedEntity` | Holds the full lifecycle of one auction slot from `RECEIVED` to a terminal state. | `BidWorkflow` | `BidLedgerView` |
| `CampaignBudgetEntity` | `EventSourcedEntity` | Tracks cumulative window spend; emits `CapBreached` when the cap is exceeded. | `BidWorkflow` | `BidLedgerView` |
| `AuctionRequestQueue` | `EventSourcedEntity` | Ingests incoming impression requests for durable fan-out. | `BidEndpoint`, `AuctionSimulator` | `AuctionRequestConsumer` |
| `AuctionRequestConsumer` | `Consumer` | Listens to `AuctionRequestQueue` events; starts one `BidWorkflow` per slot. | `AuctionRequestQueue` events | `BidWorkflow` |
| `BidLedgerView` | `View` | Read model of slots and budget state; streamed to the UI via SSE. | `AuctionSlotEntity` + `CampaignBudgetEntity` events | `BidEndpoint` |
| `AuctionSimulator` | `TimedAction` | Drips one sample auction request every 30 s. | scheduler | `AuctionRequestQueue` |
| `PerformanceSampler` | `TimedAction` | Every 5 minutes, computes win-rate and ROAS; calls `AuctionSlotEntity.recordPerformanceEval`. | scheduler | `AuctionSlotEntity` |
| `BidEndpoint` | `HttpEndpoint` | `/api/bids/*` — submit, list, SSE, campaign budget state, metadata. | — | `BidLedgerView`, `AuctionRequestQueue`, `AuctionSlotEntity`, `CampaignBudgetEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar). See `reference/data-model.md` for the full table.

### Records

```java
record AuctionRequest(String slotId, String campaignId, FloorPrice floorPrice,
                      ImpressionContext context, Instant requestedAt) {}
record FloorPrice(java.math.BigDecimal amount, String currency) {}
record ImpressionContext(String placementId, String audience, String format) {}

record BidDecision(String slotId, boolean shouldBid,
                   Optional<java.math.BigDecimal> bidPrice, String reasoning) {}

record BidResult(String slotId, boolean won,
                 Optional<java.math.BigDecimal> clearedPrice, Instant resolvedAt) {}

record PerformanceSnapshot(String campaignId, double winRate, double roas,
                           Instant windowStart, Instant windowEnd) {}

record AuctionSlot(
    String slotId,
    String campaignId,
    SlotStatus status,
    FloorPrice floorPrice,
    ImpressionContext context,
    Optional<BidDecision> decision,
    Optional<BidResult> result,
    Optional<String> haltReason,
    Optional<PerformanceSnapshot> performanceEval,
    Instant receivedAt,
    Optional<Instant> resolvedAt
) {}

record CampaignBudget(
    String campaignId,
    java.math.BigDecimal windowCapAmount,
    String currency,
    java.math.BigDecimal windowSpent,
    boolean capBreached,
    Instant windowStart,
    Optional<Instant> capBreachedAt
) {}

enum SlotStatus { RECEIVED, EVALUATING, BID_PLACED, WON, LOST, SKIPPED, HALTED, ESCALATED }
```

### Events (on `AuctionSlotEntity`)

`SlotReceived`, `BidEvaluated`, `BidPlaced`, `SlotWon`, `SlotLost`, `SlotSkipped`, `BudgetCapHalted`, `BurstEscalated`, `PerformanceEvalRecorded`.

### Events (on `CampaignBudgetEntity`)

`BudgetWindowInitialised`, `BidCharged`, `WindowReset`, `CapBreached`.

### Events (on `AuctionRequestQueue`)

`AuctionSlotSubmitted { slotId, campaignId, floorPrice, context, submittedAt }`.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/bids` — body `{ slotId, campaignId, floorPrice, context }` → `{ slotId }`. Enqueues the impression.
- `GET /api/bids` — list all slots. Optional `?status=RECEIVED|EVALUATING|BID_PLACED|WON|LOST|SKIPPED|HALTED|ESCALATED`.
- `GET /api/bids/{slotId}` — one slot.
- `GET /api/bids/sse` — server-sent events stream of every slot change.
- `GET /api/campaigns/{campaignId}/budget` — current `CampaignBudget` state.
- `POST /api/campaigns/{campaignId}/budget/reset` — reset the window spend (test/ops use).
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Real-Time Bid Arbitration Agent"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with syntax-highlighted Java snippets. Mermaid state-label CSS overrides and theme variables per Lesson 24.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit an auction request, campaign budget bar, live slot ledger streamed via SSE with status pills (RECEIVED / EVALUATING / BID_PLACED / WON / LOST / SKIPPED / HALTED / ESCALATED), expand-row to see bid decision, result, and performance eval.

Browser title: `<title>Akka Sample: Real-Time Bid Arbitration Agent</title>`. Tab switching is attribute-based (`data-tab` / `data-panel`), never NodeList index (Lesson 26); exactly five `.tab-panel` sections in the DOM, no zombie panels.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **H1 — budget-cap halt** (`halt · budget-cap`): `CampaignBudgetEntity` emits `CapBreached` when window spend exceeds the configured cap; `BidWorkflow` checks budget state before invoking `BidArbitrationAgent` and routes slots to `HALTED`. Blocking.
- **E1 — scheduled performance eval** (`eval-periodic · scheduled-eval`): `PerformanceSampler` (TimedAction) computes win-rate and ROAS every 5 minutes and writes a `PerformanceEvalRecorded` event to any slot that was resolved in the window. Non-blocking.
- **S1 — burst escalation HITL** (`hitl · application`): `BidWorkflow` counts bid placements in a rolling 60-second window; when the count exceeds the burst threshold, the slot moves to `ESCALATED` and an ops-queue message is written for human review. Blocking for that slot.

## 9. Agent prompts

- `BidArbitrationAgent` → `prompts/bid-arbitration-agent.md`. Evaluates impression context; returns `BidDecision`.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit an auction request; slot progresses `RECEIVED → EVALUATING → BID_PLACED → WON` or `LOST` within 30 s; UI reflects each transition via SSE.
2. **J2** — Exhaust the campaign budget window; subsequent slots enter `HALTED` without calling the agent.
3. **J3** — Trigger a bid burst (submit > threshold impressions in 60 s); affected slots enter `ESCALATED`.
4. **J4** — Wait for `PerformanceSampler`; resolved slots gain a `PerformanceEvalRecorded` annotation visible in the App UI.
5. **J5** — Simulator drips requests; the ledger is non-empty on first load.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named real-time-bid-arbitration demonstrating the
market-making-auction × sales-marketing cell. Runs out of the box (no
external services). Maven group io.akka.samples. Maven artifact
market-making-auction-sales-marketing-real-time-bid-arbitration.
Java package io.akka.samples.realtimebidarbitrationagent. Akka 3.6.0. HTTP port 9441.

Components to wire (exactly):
- 1 AutonomousAgent:
  * BidArbitrationAgent — definition() with capability(TaskAcceptance.of(EVALUATE_IMPRESSION)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/bid-arbitration-agent.md.
    Returns BidDecision{slotId, shouldBid, Optional<BigDecimal> bidPrice, reasoning} for
    EVALUATE_IMPRESSION.

- 1 Workflow BidWorkflow with steps:
  budgetCheckStep -> evaluateStep -> placeStep -> resultStep -> chargeStep.
  budgetCheckStep reads CampaignBudgetEntity.getBudget; if capBreached, transition to haltStep
  which calls AuctionSlotEntity.haltSlot and ends. evaluateStep calls
  forAutonomousAgent(BidArbitrationAgent.class, EVALUATE_IMPRESSION) with stepTimeout(ofSeconds(20)).
  If shouldBid is false, transition to skipStep which calls AuctionSlotEntity.skipSlot and ends.
  placeStep calls the simulated exchange (in-process) and calls AuctionSlotEntity.placeBid.
  A burst counter in BidWorkflow (InMemoryCounter, keyed by campaignId + minute bucket) increments
  on each placeStep; if the count exceeds burstThreshold (configurable, default 20 per 60 s),
  transition to escalateStep which calls AuctionSlotEntity.escalateSlot and writes a message to
  OpsQueue before ending. resultStep calls AuctionSlotEntity.recordResult. chargeStep calls
  CampaignBudgetEntity.chargeBid(bidPrice). WorkflowSettings is nested inside Workflow — no import.

- 2 EventSourcedEntities:
  * AuctionSlotEntity holding state AuctionSlot{slotId, campaignId, SlotStatus, FloorPrice,
    ImpressionContext, Optional<BidDecision> decision, Optional<BidResult> result,
    Optional<String> haltReason, Optional<PerformanceSnapshot> performanceEval,
    Instant receivedAt, Optional<Instant> resolvedAt}.
    SlotStatus enum: RECEIVED, EVALUATING, BID_PLACED, WON, LOST, SKIPPED, HALTED, ESCALATED.
    Events: SlotReceived, BidEvaluated, BidPlaced, SlotWon, SlotLost, SlotSkipped,
    BudgetCapHalted, BurstEscalated, PerformanceEvalRecorded.
    Commands: receiveSlot, evaluateSlot, placeBid, recordResult, skipSlot, haltSlot,
    escalateSlot, recordPerformanceEval, getSlot.
    emptyState() returns AuctionSlot.initial("", "") with no commandContext() reference.
  * CampaignBudgetEntity holding state CampaignBudget{campaignId, BigDecimal windowCapAmount,
    String currency, BigDecimal windowSpent, boolean capBreached, Instant windowStart,
    Optional<Instant> capBreachedAt}.
    Events: BudgetWindowInitialised, BidCharged, WindowReset, CapBreached.
    Commands: initialiseBudget, chargeBid, resetWindow, getBudget.

- 1 EventSourcedEntity AuctionRequestQueue with command submitSlot(slotId, campaignId,
  floorPrice, context) emitting AuctionSlotSubmitted{slotId, campaignId, floorPrice, context,
  submittedAt}.

- 1 View BidLedgerView with row type BidLedgerRow (mirrors AuctionSlot minus heavy nested
  payloads; every nullable field is Optional<T>). Table updater consumes AuctionSlotEntity events.
  ONE query getAllSlots SELECT * AS slots FROM bid_ledger_view. No WHERE status filter
  (Akka cannot auto-index enum columns) — caller filters client-side.

- 1 Consumer AuctionRequestConsumer subscribed to AuctionRequestQueue events; on
  AuctionSlotSubmitted starts a BidWorkflow with the slotId as the workflow id.

- 2 TimedActions:
  * AuctionSimulator — every 30 s, reads next line from
    src/main/resources/sample-events/auction-requests.jsonl and calls
    AuctionRequestQueue.submitSlot.
  * PerformanceSampler — every 5 minutes, queries BidLedgerView.getAllSlots, groups resolved
    slots (WON, LOST) by campaignId for the current window, computes win-rate and ROAS, then
    calls AuctionSlotEntity.recordPerformanceEval for each resolved slot in that window.

- 2 HttpEndpoints:
  * BidEndpoint at /api with POST /bids, GET /bids, GET /bids/{slotId},
    GET /bids/sse, GET /campaigns/{campaignId}/budget,
    POST /campaigns/{campaignId}/budget/reset, and the /api/metadata/* endpoints
    serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- BidTasks.java declaring Task<R> constant: EVALUATE_IMPRESSION (BidDecision).
- Domain records AuctionRequest, FloorPrice, ImpressionContext, BidDecision, BidResult,
  PerformanceSnapshot.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9441 and
  akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6), openai
  (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
  Campaign budget defaults: window-cap-amount = 500.00, currency = USD,
  burst-threshold = 20, window-duration-seconds = 3600.
- src/main/resources/sample-events/auction-requests.jsonl with 8 canned impression lines.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls (H1 budget-cap halt,
  E1 eval-periodic scheduled-eval, S1 hitl application) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector = ["programmatic advertising",
  "real-time bidding"], decisions.authority_level = automated-within-bounds,
  data.pii = false, capabilities.content_generation = false; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/bid-arbitration-agent.md loaded at agent startup as the system prompt.
- README.md at the project root: title "Akka Sample: Real-Time Bid Arbitration Agent",
  one-line pitch, prerequisites, generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
- src/main/resources/static-resources/index.html — a single self-contained HTML file
  (no ui/ folder, no npm build). Five tabs: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand rows), App UI
  (form + campaign budget bar + live slot ledger with status pills). Browser title exactly:
  <title>Akka Sample: Real-Time Bid Arbitration Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options
  via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
        Mock shapes for BidArbitrationAgent:
          bid-arbitration-agent.json — 6–8 BidDecision entries with a mix of
          shouldBid=true (bidPrice between floor and 2x floor) and
          shouldBid=false, each with a one-sentence reasoning.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent; extends clause matches verbatim.
- Lesson 4: every workflow step that calls the agent gets an explicit stepTimeout(ofSeconds(20));
  default 5 s is too short for LLM calls.
- Lesson 6: Optional<T> for every nullable field on the View row record.
- Lesson 7: AutonomousAgent requires the companion BidTasks.java declaring Task<R> constants.
- Lesson 8: verify model names current before locking (claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash).
- Lesson 9: run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: explicit dev-mode.http-port = 9441 in application.conf.
- Lesson 11: never render source.platform anywhere user-facing.
- Lesson 12: UI fits the 1080px content column with no horizontal scroll.
- Lesson 13: integration label is descriptive ("Runs out of the box"), never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in any user-facing surface.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides AND theme variables
  (state-diagram label colour, edge-label foreignObject overflow:visible,
  transitionLabelColor #cccccc). Without these, state names render black-on-black.
- Lesson 25: five-option key sourcing; never write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, never NodeList index;
  delete removed panels from the DOM, do not display:none them.
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
