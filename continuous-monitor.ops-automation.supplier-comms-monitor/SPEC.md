# SPEC — supplier-comms-monitor

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Supplier Communications Agent.
**One-line pitch:** A background worker watches open purchase orders, assesses delivery risk, drafts supplier outreach, and holds every material change for buyer approval before sending.

## 2. What this blueprint demonstrates

The **continuous-monitor** coordination pattern wired with three governance mechanisms layered on top of two AI primitives (`DeliveryRiskAgent` and `SupplierOutreachAgent`). Specifically:

- A **before-tool-call guardrail** on the (simulated) `sendSupplierEmail` tool enforces "outreach drafts must stay drafts until the guardrail confirms send eligibility". Because supplier emails are external communications, the guardrail verifies the PO's current state before any message leaves the process.
- A **HITL approve-before-send** gate is the production safeguard for material PO changes: the agent never sends an outreach that modifies a delivery date or quantity commitment; only an authenticated buyer can transition that PO to OUTREACH_SENT.
- An **eval-periodic** accuracy monitor running every 60 minutes compares the `DeliveryRiskAgent`'s predictions on now-closed POs against their actual delivery outcomes, producing a continuous precision signal.

The result is a system where no external supplier message goes out without both guardrail clearance and — for consequential changes — explicit buyer sign-off, while accuracy is monitored continuously to catch model drift.

## 3. User-facing flows

The user opens the App UI tab.

1. The system shows the live PO board: every open purchase order, its assessed risk tier, and (if an outreach was drafted) the proposed supplier message.
2. `PoOrderPoller` (TimedAction) ticks every 20 s and ingests new or updated POs from the simulated feed into `PoFeedQueue`.
3. For each new PO event: `PoFeedConsumer` publishes it onto `PurchaseOrderEntity`, then `PoOrderWorkflow` starts and calls `DeliveryRiskAgent` to classify risk.
4. If the classification is `AT_RISK` or `CRITICAL`, `SupplierOutreachAgent` drafts a confirmation request. POs flagged `CRITICAL` or those with quantity/date changes land in `AWAITING_BUYER_APPROVAL`.
5. The buyer clicks Approve in the UI — the outreach transitions to OUTREACH_SENT (simulated; no real email leaves).
6. The buyer clicks Escalate — the PO transitions to PROCUREMENT_REVIEW for a procurement specialist.
7. `AccuracyEvalRunner` (TimedAction) ticks every 60 minutes, picks recently closed POs, compares their predicted risk classification to the actual on-time/late outcome, and writes an `AccuracyEvaluated` event with a precision score.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PoOrderPoller` | `TimedAction` | Drips simulated PO feed events every 20 s. | scheduler | `PoFeedQueue` |
| `PoFeedQueue` | `EventSourcedEntity` | Append-only log of `PoFeedReceived` events. | `PoOrderPoller`, `PoOrderEndpoint` | `PoFeedConsumer` |
| `PoFeedConsumer` | `Consumer` | Reads `PoFeedReceived` events, registers each PO onto `PurchaseOrderEntity`, starts `PoOrderWorkflow`. | `PoFeedQueue` events | `PurchaseOrderEntity`, `PoOrderWorkflow` |
| `DeliveryRiskAgent` | `Agent` (typed, not autonomous) | Classifies PO into `ON_TRACK`, `AT_RISK`, or `CRITICAL`. | invoked by Workflow | returns `RiskAssessment` |
| `SupplierOutreachAgent` | `AutonomousAgent` | Drafts a supplier confirmation request for AT_RISK / CRITICAL POs. | invoked by Workflow | returns `OutreachDraft` |
| `PoOrderWorkflow` | `Workflow` | Per-PO orchestration: assess → (maybe draft) → guardrail check → await buyer if material → send / escalate. | `PoFeedConsumer` (one workflow per PO) | `PurchaseOrderEntity` |
| `PurchaseOrderEntity` | `EventSourcedEntity` | Lifecycle per PO: opened → risk-assessed → outreach-drafted → awaiting-buyer / outreach-sent / escalated. | `PoOrderWorkflow` | `PoOrderView` |
| `PoOrderView` | `View` | Read-model row per PO for the UI. | `PurchaseOrderEntity` events | `PoOrderEndpoint` |
| `AccuracyEvalRunner` | `TimedAction` | Every 60 min, samples closed POs; calls accuracy judge; writes `AccuracyEvaluated`. | scheduler | `PurchaseOrderEntity` |
| `PoOrderEndpoint` | `HttpEndpoint` | `/api/orders/*` — list, get, approve, escalate, SSE. | — | `PoOrderView`, `PurchaseOrderEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

```java
record PoFeedEvent(String poId, String supplierId, String supplierName,
                   String itemDescription, int orderedQty, LocalDate promisedDate,
                   String currency, BigDecimal unitPrice, Instant feedTimestamp) {}

record RiskAssessment(RiskTier tier, String confidence, String rationale,
                      List<String> riskFactors) {}
enum RiskTier { ON_TRACK, AT_RISK, CRITICAL }

record OutreachDraft(String subjectLine, String body,
                     boolean requiresBuyerApproval, Instant draftedAt) {}

record BuyerDecision(boolean approved, String decidedBy,
                     Optional<String> escalationNote, Instant decidedAt) {}

record AccuracyScore(String poId, RiskTier predictedTier, String actualOutcome,
                     boolean predictionCorrect, String evalRationale) {}

record PurchaseOrder(
    String poId,
    PoFeedEvent feedEvent,
    Optional<RiskAssessment> riskAssessment,
    Optional<OutreachDraft> outreachDraft,
    Optional<BuyerDecision> buyerDecision,
    Optional<AccuracyScore> accuracyScore,
    PoStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum PoStatus {
    OPENED, RISK_ASSESSED, OUTREACH_DRAFTED, AWAITING_BUYER_APPROVAL,
    OUTREACH_SENT, PROCUREMENT_REVIEW, ON_TRACK_CONFIRMED, CLOSED
}
```

Events on `PurchaseOrderEntity`: `PoOpened`, `RiskAssessed`, `OutreachDrafted`, `BuyerApproved`, `BuyerEscalated`, `OutreachSent`, `ProcurementReview`, `OnTrackConfirmed`, `PoClosed`, `AccuracyEvaluated`.

Events on `PoFeedQueue`: `PoFeedReceived` (the raw feed event as the audit log).

See `reference/data-model.md`.

## 6. API contract

- `GET /api/orders` — list all POs. Optional `?status=…`.
- `GET /api/orders/{id}` — one PO.
- `POST /api/orders/{id}/approve` — body `{ decidedBy }` → transitions AWAITING_BUYER_APPROVAL to OUTREACH_SENT.
- `POST /api/orders/{id}/escalate` — body `{ decidedBy, escalationNote }` → transitions to PROCUREMENT_REVIEW.
- `GET /api/orders/sse` — Server-Sent Events for every PO change.
- `GET /api/metadata/{readme,risk-survey,eval-matrix,classification-questions,survey-template}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas.

## 7. UI

Five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Supplier Communications Agent</title>`.

App UI tab is the most distinctive: it shows the **live PO board** with a status column layout. Approve/Escalate are click-buttons inside each card; bulk actions are out of scope.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** on the `sendSupplierEmail` simulated tool: blocks the call unless the PO's current state satisfies send eligibility (guardrail reads entity state before allowing the tool to proceed).
- **H1 — HITL approve-before-send** gate: for POs where `outreachDraft.requiresBuyerApproval == true`, the workflow explicitly pauses in AWAITING_BUYER_APPROVAL; only `PoOrderEndpoint`'s approve/escalate endpoints can advance it.
- **E1 — eval-periodic** (every 60 minutes): computes delivery-prediction accuracy on closed POs by comparing `predictedTier` against `actualOutcome`.

## 9. Agent prompts

- `DeliveryRiskAgent` → `prompts/delivery-risk.md`. Typed classifier. Always returns one of the three `RiskTier` values.
- `SupplierOutreachAgent` → `prompts/supplier-outreach.md`. Drafts a supplier-facing confirmation request based on the PO record. Never sends directly.
- `AccuracyJudge` (used by `AccuracyEvalRunner`) → `prompts/accuracy-judge.md`. Scores the original risk prediction against the actual delivery outcome and writes a precision signal.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Simulator drops a PO feed event; it appears in the UI within 20 s; passes assess → draft outreach → AWAITING_BUYER_APPROVAL.
2. **J2** — Buyer clicks Approve; PO transitions to OUTREACH_SENT; no real network call leaves the process.
3. **J3** — Buyer clicks Escalate with a note; PO transitions to PROCUREMENT_REVIEW with the note visible.
4. **J4** — AccuracyEvalRunner scores at least one closed PO within 60 minutes; the precision score appears on the card.
5. **J5** — An ON_TRACK PO auto-confirms without buyer interaction.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named supplier-comms-monitor demonstrating the continuous-monitor ×
ops-automation cell. Runs out of the box (in-memory PO feed simulator; no real
Dynamics 365 integration). Maven group io.akka.samples. Artifact
continuous-monitor-ops-automation-supplier-comms-monitor. Java package
io.akka.samples.suppliercommunicationsagent. HTTP port 9831.

Components to wire (exactly):

- 1 Agent (typed, NOT autonomous) DeliveryRiskAgent — risk classifier. System prompt loaded
  from prompts/delivery-risk.md. Input: PoFeedEvent{poId, supplierId, supplierName,
  itemDescription, orderedQty, promisedDate, currency, unitPrice, feedTimestamp}. Output:
  RiskAssessment{tier: RiskTier (enum ON_TRACK/AT_RISK/CRITICAL), confidence:
  "high"|"medium"|"low", rationale: String, riskFactors: List<String>}. Defaults to AT_RISK
  under uncertainty.

- 1 AutonomousAgent SupplierOutreachAgent — definition() with
  capability(TaskAcceptance.of(DRAFT_OUTREACH).maxIterationsPerTask(3)). System prompt from
  prompts/supplier-outreach.md. Input: PoFeedEvent + RiskAssessment. Output:
  OutreachDraft{subjectLine, body, requiresBuyerApproval: boolean, draftedAt}. The agent
  NEVER calls sendSupplierEmail directly.

- 1 AutonomousAgent AccuracyJudge — definition() with
  capability(TaskAcceptance.of(EVAL_ACCURACY).maxIterationsPerTask(2)). System prompt from
  prompts/accuracy-judge.md. Input: PurchaseOrder (at close) + actualOutcome: String.
  Output: AccuracyScore{poId, predictedTier, actualOutcome, predictionCorrect, evalRationale}.

- 1 Workflow PoOrderWorkflow per PO with steps: assessStep -> conditional branch:
  ON_TRACK -> confirmStep (ends with OnTrackConfirmed); AT_RISK -> draftStep ->
  requiresBuyerApproval? -> awaitBuyerStep -> finaliseStep;
  CRITICAL -> draftStep -> awaitBuyerStep -> finaliseStep.
  assessStep wraps DeliveryRiskAgent with WorkflowSettings.builder()
  .stepTimeout(Duration.ofSeconds(15)). draftStep wraps SupplierOutreachAgent with
  stepTimeout 30s. awaitBuyerStep polls PurchaseOrderEntity.getOrder every 5s; on
  decision.isPresent() advances. No auto-timeout on awaitBuyer — POs wait indefinitely.
  finaliseStep emits OutreachSent or ProcurementReview based on decision.approved.

- 2 EventSourcedEntities:
  * PoFeedQueue — append-only audit log of inbound PO feed events. Command
    receive(PoFeedEvent) emits PoFeedReceived{feedEvent}.
  * PurchaseOrderEntity (one per poId) — full per-PO lifecycle. State PurchaseOrder{poId,
    feedEvent: PoFeedEvent{poId, supplierId, supplierName, itemDescription, orderedQty,
    promisedDate, currency, unitPrice, feedTimestamp}, Optional<RiskAssessment> riskAssessment,
    Optional<OutreachDraft> outreachDraft, Optional<BuyerDecision> buyerDecision (with
    approved: boolean, decidedBy: String, Optional<String> escalationNote, decidedAt: Instant),
    Optional<AccuracyScore> accuracyScore, PoStatus status, Instant createdAt,
    Optional<Instant> closedAt}. PoStatus enum: OPENED, RISK_ASSESSED, OUTREACH_DRAFTED,
    AWAITING_BUYER_APPROVAL, OUTREACH_SENT, PROCUREMENT_REVIEW, ON_TRACK_CONFIRMED, CLOSED.
    Events: PoOpened, RiskAssessed, OutreachDrafted, BuyerApproved, BuyerEscalated,
    OutreachSent, ProcurementReview, OnTrackConfirmed, PoClosed, AccuracyEvaluated.
    Commands: registerPo, attachRiskAssessment, attachOutreachDraft, recordBuyerApproval,
    recordBuyerEscalation, markOutreachSent, markProcurementReview, markOnTrackConfirmed,
    closePo, recordAccuracy, getOrder. emptyState() returns PurchaseOrder.initial("", null)
    without commandContext() reference.

- 1 Consumer PoFeedConsumer subscribed to PoFeedQueue events; for each PoFeedReceived,
  calls PurchaseOrderEntity.registerPo with the feed event, then starts a PoOrderWorkflow
  with poId as the workflow id.

- 1 View PoOrderView with row type PoOrderRow (mirrors PurchaseOrder). Table updater
  consumes PurchaseOrderEntity events. ONE query getAllOrders SELECT * AS orders FROM
  po_order_view. No WHERE status filter — caller filters client-side.

- 2 TimedActions:
  * PoOrderPoller — every 20s, reads next line from
    src/main/resources/sample-events/po-feed-events.jsonl and calls PoFeedQueue.receive.
  * AccuracyEvalRunner — every 60 minutes, queries PoOrderView.getAllOrders, picks up to 5
    CLOSED POs without an accuracyScore (oldest-first), calls AccuracyJudge, then calls
    PurchaseOrderEntity.recordAccuracy per PO.

- 2 HttpEndpoints:
  * PoOrderEndpoint at /api with GET /orders, GET /orders/{id}, POST /orders/{id}/approve
    (body {decidedBy}), POST /orders/{id}/escalate (body {decidedBy, escalationNote}),
    GET /orders/sse, and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/. Approve writes BuyerApproved + OutreachSent; escalate
    writes BuyerEscalated + ProcurementReview.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

Companion files:
- PoTasks.java declaring three Task<R> constants: DRAFT_OUTREACH (OutreachDraft),
  EVAL_ACCURACY (AccuracyScore), and ASSESS_RISK (RiskAssessment).
- Domain records PoFeedEvent, RiskAssessment, OutreachDraft, BuyerDecision, AccuracyScore.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9831 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars.
- src/main/resources/sample-events/po-feed-events.jsonl with 8 canned PO lines covering
  ON_TRACK (standard delivery, lead time comfortable), AT_RISK (late confirmation, single
  source), and CRITICAL (promised date overdue, no supplier response) classifications.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 3 controls: G1 guardrail before-tool-call,
  H1 hitl application, E1 eval-periodic performance-monitor. Matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root with decisions.authority_level = draft-only,
  oversight.human_in_loop = true, oversight.reviewer_must_approve_material_changes = true,
  failure.failure_modes including "unauthorized-external-supplier-email" and
  "inaccurate-delivery-prediction"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/delivery-risk.md, prompts/supplier-outreach.md, prompts/accuracy-judge.md loaded
  as agent system prompts.
- README.md at the project root: title "Akka Sample: Supplier Communications Agent",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section.
- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar with the App UI tab using a two-column layout
  (left = live PO board with risk tier chips and status pills; right = selected PO detail
  with outreach draft textarea + Approve/Escalate buttons). Browser title exactly:
  <title>Akka Sample: Supplier Communications Agent</title>. No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent.
        Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java implementing the ModelProvider interface
  with a per-agent dispatch on the agent class or Task<R> id. Each branch
  reads src/main/resources/mock-responses/<agent>.json, picks one entry
  pseudo-randomly per call, and deserialises into the agent's typed return.
- Per-agent mock-response shapes for THIS blueprint:
    delivery-risk.json — 8–12 RiskAssessment entries spanning
      ON_TRACK (recent acknowledgment, lead time comfortable),
      AT_RISK (no supplier acknowledgment within expected window, single
      source), and CRITICAL (promised date is today or past, no response).
      Each entry has confidence and a rationale sentence plus 1–3 riskFactors.
      The mock should default to AT_RISK under ambiguity.
    supplier-outreach.json — 4–6 OutreachDraft entries with subjectLine ≤ 70
      chars starting "Delivery Confirmation:", body 2–4 paragraphs, signed
      "— Procurement Operations". requiresBuyerApproval=true for all CRITICAL
      drafts; false for routine AT_RISK acknowledgment requests.
    accuracy-judge.json — 6–8 AccuracyScore entries with predictionCorrect
      true/false and one-sentence evalRationale citing the strongest signal.
- A `MockModelProvider.seedFor(poId)` helper makes per-PO selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Optional<T> for every nullable field on a View row record.
- WorkflowSettings is nested inside Workflow — no import needed.
- emptyState() never calls commandContext().
- AutonomousAgent never silently downgraded to Agent.
- The agent NEVER sends; only the buyer Approve endpoint can transition to OUTREACH_SENT.
- The generated static-resources/index.html must include the mermaid CSS
  overrides AND theme variables from Lesson 24 (state-diagram label colour,
  edge-label foreignObject overflow:visible, transitionLabelColor #cccccc).
- Tab switching in static-resources/index.html MUST match by data-tab /
  data-panel attribute, NEVER by NodeList index (Lesson 26).
- The Overview tab's Try-it card shows just "/akka:build".
- No forbidden words in user-facing text.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous.
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
