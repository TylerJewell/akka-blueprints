# SPEC — ad-spend-payment-agent

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** AI Ads Generation Agent with Crypto Payments.
**One-line pitch:** Submit a campaign brief; a Campaign Planner drafts ad creatives on a creative ledger, routes each approved placement through a human spend-approval gate, then fires an on-chain payment via a crypto-payment tool — with a guardrail that blocks any payment above the campaign budget and an operator halt that stops new payment dispatches instantly.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern where two specialist agents — a Copywriter and a PaymentExecutor — carry out the steps the planner schedules. The Campaign Planner owns two ledgers: a **creative ledger** (campaign facts, missing information, the ordered creative plan, the current dispatch) and a **payment ledger** (each placement attempt, approval status, on-chain tx hash or failure reason). On each loop tick the planner reads both ledgers, picks the next step, and either continues, replans, halts, or completes.

The blueprint demonstrates three governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets every payment dispatch against the campaign budget cap and a payment-policy allow-list before the on-chain call fires,
- a **human-in-the-loop approval gate** that pauses the workflow until a named approver explicitly authorises each placement spend via the UI,
- an **operator halt** (`halt`, flavor `operator-regulator-stop`) that stops new payment dispatches immediately — intended for emergency regulatory or treasury situations where an operator or external regulator must freeze on-chain activity.

## 3. User-facing flows

The user opens the App UI tab and submits a campaign brief via the form.

1. The system creates a `Campaign` record in `PLANNING` and starts a `CampaignWorkflow`.
2. The Planner drafts a `CreativeLedger { facts, missing, plan, dispatch }` and emits `CampaignPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - Planner reads both ledgers and proposes a `DispatchDecision { executor, task, rationale }`.
   - The **before-payment guardrail** vets payment decisions: checks the proposed spend against the remaining budget cap and validates the destination wallet against the placement allow-list. On rejection the workflow records a `TaskBlocked` entry and asks the Planner to revise.
   - If the executor is `COPYWRITER`, the workflow calls `CopywriterAgent` directly.
   - If the executor is `PAYMENT`, the workflow pauses at the **HITL approval gate**: it writes an `ApprovalEntity` record, emits `PaymentApprovalRequested`, and waits. An approver clicks Approve or Reject in the UI; the entity emits `ApprovalGranted` or `ApprovalRejected` and the workflow resumes.
   - The chosen executor runs the task and returns a typed result.
   - The **secret sanitizer** scrubs the result for wallet keys and seed phrases before the content lands on the payment ledger or reaches the planner's next prompt.
   - The workflow appends a `PlacementEntry` or `CreativeEntry` to the appropriate ledger.
4. The Planner decides on each loop tick: `CONTINUE`, `REPLAN`, `COMPLETE`, or `FAIL`.
5. On `COMPLETE`, the Planner produces a `CampaignReport { summary, placements, totalSpentWei }` and emits `CampaignCompleted`.
6. An operator can press **Halt new dispatches** at any time. The workflow finishes the in-flight copywriter task or pending approval, then ends with `CampaignHaltedOperator`. The Campaign moves to `HALTED`.

A `BriefSimulator` (TimedAction) drips a sample campaign brief every 90 s so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CampaignPlannerAgent` | `AutonomousAgent` | Plans, dispatches, and decides on each loop tick. Maintains the creative ledger; reads the payment ledger. Produces `CampaignReport` on completion. | `CampaignWorkflow` | returns typed result to workflow |
| `CopywriterAgent` | `AutonomousAgent` | Drafts or refines ad copy. Returns a typed `CreativeResult`. | `CampaignWorkflow` | — |
| `PaymentExecutorAgent` | `AutonomousAgent` | Constructs and submits a simulated on-chain payment instruction. Returns a `PaymentResult` with a tx hash or failure reason. | `CampaignWorkflow` | — |
| `CampaignWorkflow` | `Workflow` | Drives the plan → dispatch-guarded → execute → sanitize → record → approval-gate → decide loop, plus replan and halt branches. | `CampaignEndpoint`, `CampaignRequestConsumer` | `CampaignEntity`, `ApprovalEntity` |
| `CampaignEntity` | `EventSourcedEntity` | Holds the campaign lifecycle, creative ledger, payment ledger, and final report. | `CampaignWorkflow` | `CampaignView` |
| `ApprovalEntity` | `EventSourcedEntity` | Holds one pending spend-approval record per placement. Keyed by `(campaignId, placementId)`. Commands: `requestApproval`, `grant`, `reject`, `get`. | `CampaignWorkflow`, `CampaignEndpoint` | `CampaignWorkflow` (workflow polls) |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `CampaignEndpoint` (operator action) | `CampaignWorkflow` (polls) |
| `CampaignQueue` | `EventSourcedEntity` | Audit log of submitted campaign briefs. | `CampaignEndpoint`, `BriefSimulator` | `CampaignRequestConsumer` |
| `CampaignView` | `View` | List-of-campaigns read model for the UI. | `CampaignEntity` events | `CampaignEndpoint` |
| `CampaignRequestConsumer` | `Consumer` | Subscribes to `CampaignQueue` events; starts a `CampaignWorkflow` per submission. | `CampaignQueue` events | `CampaignWorkflow` |
| `BriefSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/brief-prompts.jsonl` and enqueues it. | scheduler | `CampaignQueue` |
| `StaleCampaignMonitor` | `TimedAction` | Every 60 s, marks any campaign stuck in `EXECUTING` or `AWAITING_APPROVAL` past 10 minutes as `STUCK`. | scheduler | `CampaignEntity` |
| `CampaignEndpoint` | `HttpEndpoint` | `/api/campaigns/*` — submit, get, list, SSE, approve/reject placement, operator halt. | — | `CampaignView`, `CampaignQueue`, `CampaignEntity`, `ApprovalEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record CampaignBrief(
    String title,
    String brandVoice,
    String targetAudience,
    String adFormat,
    long budgetWei,
    String requestedBy
) {}

record CreativeLedger(
    List<String> facts,
    List<String> missing,
    List<String> plan,
    Optional<DispatchDecision> currentDispatch
) {}

record DispatchDecision(
    ExecutorKind executor,
    String task,
    String rationale,
    Optional<Long> proposedSpendWei
) {}

record CreativeResult(
    ExecutorKind executor,
    String task,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record CreativeEntry(
    int attempt,
    ExecutorKind executor,
    String task,
    CreativeVerdict verdict,
    String sanitizedContent,
    Optional<String> blocker,
    Instant recordedAt
) {}

record PaymentInstruction(
    String placementId,
    String destinationWallet,
    long amountWei,
    String token,
    String memo
) {}

record PaymentResult(
    String placementId,
    boolean ok,
    Optional<String> txHash,
    Optional<String> failureReason,
    Instant executedAt
) {}

record PlacementEntry(
    String placementId,
    String task,
    ApprovalStatus approvalStatus,
    Optional<String> approvedBy,
    Optional<Instant> approvedAt,
    Optional<PaymentResult> paymentResult,
    Instant recordedAt
) {}

record PaymentLedger(List<PlacementEntry> entries) {}

record CampaignReport(
    String summary,
    List<PlacementEntry> placements,
    long totalSpentWei,
    Instant producedAt
) {}

record Campaign(
    String campaignId,
    String title,
    String requestedBy,
    long budgetWei,
    CampaignStatus status,
    Optional<CreativeLedger> creativeLedger,
    Optional<PaymentLedger> paymentLedger,
    Optional<CampaignReport> report,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

record ApprovalRequest(
    String campaignId,
    String placementId,
    String task,
    long amountWei,
    String destinationWallet,
    String token,
    ApprovalStatus status,
    Optional<String> approvedBy,
    Optional<Instant> decidedAt
) {}

enum ExecutorKind { COPYWRITER, PAYMENT }
enum CreativeVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED }
enum ApprovalStatus { PENDING, APPROVED, REJECTED }
enum CampaignStatus { PLANNING, EXECUTING, AWAITING_APPROVAL, COMPLETED, FAILED, HALTED, STUCK }
```

### Events (`CampaignEntity`)

`CampaignCreated`, `CampaignPlanned`, `CreativeDispatched`, `CreativeBlocked`, `CreativeRecorded`, `PaymentApprovalRequested`, `PaymentApproved`, `PaymentRejected`, `PaymentRecorded`, `LedgerRevised`, `CampaignCompleted`, `CampaignFailed`, `CampaignHaltedOperator`, `CampaignFailedTimeout`.

### Events (`ApprovalEntity`)

`ApprovalRequested { campaignId, placementId, amountWei, destinationWallet, token, requestedAt }`, `ApprovalGranted { approvedBy, grantedAt }`, `ApprovalRejected { rejectedBy, reason, rejectedAt }`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`CampaignQueue`)

`BriefSubmitted { campaignId, title, budgetWei, requestedBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/campaigns` — body `CampaignBrief` → `202 { campaignId }`. Starts a workflow.
- `GET /api/campaigns` — list all campaigns. Optional `?status=...`.
- `GET /api/campaigns/{id}` — one campaign (full ledgers + report).
- `GET /api/campaigns/sse` — server-sent events stream of every campaign change.
- `GET /api/campaigns/{id}/approvals` — list pending placement approvals for this campaign.
- `POST /api/campaigns/{id}/approvals/{placementId}/approve` — body `{ approvedBy }` → `200`.
- `POST /api/campaigns/{id}/approvals/{placementId}/reject` — body `{ rejectedBy, reason }` → `200`.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "AI Ads Generation Agent with Crypto Payments"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a campaign brief (title, brand voice, audience, ad format, budget), a pending-approvals pane showing placements awaiting human sign-off with Approve/Reject buttons, the operator halt/resume control, and a live list of campaigns with status pills; expand a row to see the creative ledger, payment ledger entries, and the final report.

Browser title: `<title>Akka Sample: AI Ads Generation Agent with Crypto Payments</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-payment guardrail** (`before-tool-call` on `CampaignPlannerAgent`): every `DispatchDecision` with `executor = PAYMENT` is checked against (a) the remaining campaign budget cap (`proposedSpendWei ≤ remainingBudgetWei`), (b) a destination-wallet allow-list loaded from `sample-data/wallet-allowlist.json`, (c) a supported-token list (only `ETH` and `USDC` accepted). Blocking. On rejection the workflow records a `CreativeBlocked` entry and asks the Planner to revise.
- **HI1 — human spend-approval gate** (`hitl`, flavor `application`): every payment dispatch is paused until a named approver acts via `POST /api/campaigns/{id}/approvals/{placementId}/approve`. The workflow writes an `ApprovalEntity`, emits `PaymentApprovalRequested`, and suspends until it receives `ApprovalGranted` or `ApprovalRejected`. On rejection the planner replans.
- **HO1 — operator/regulator halt** (`halt`, flavor `operator-regulator-stop`): the operator dashboard's Halt button writes to `SystemControlEntity` via `POST /api/control/halt`. The workflow polls this entity before each payment dispatch and exits with `CampaignHaltedOperator` if the flag is set. Copywriter work in-flight finishes; no new payment calls fire.

## 9. Agent prompts

- `CampaignPlannerAgent` → `prompts/campaign-planner.md`. Maintains both ledgers; decides next step.
- `CopywriterAgent` → `prompts/copywriter.md`. Drafts or refines ad copy.
- `PaymentExecutorAgent` → `prompts/payment-executor.md`. Constructs and submits the on-chain payment instruction.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Launch a banner ad for AkkaStore with a 0.05 ETH budget targeting developers." Campaign progresses `PLANNING → EXECUTING → AWAITING_APPROVAL → EXECUTING → COMPLETED` within ~4 minutes. The approval pane shows the pending placement; clicking Approve unblocks the workflow; the campaign report shows a non-zero `totalSpentWei`.
2. **J2** — Submit a brief with a 0.001 ETH budget; the planner proposes a spend of 0.002 ETH for the placement. The guardrail blocks the payment dispatch with `BLOCKED_BY_GUARDRAIL`; the planner either revises to a smaller placement or the campaign ends in `FAILED`.
3. **J3** — Submit a brief, approve one placement, then click **Halt new dispatches** while the campaign is `EXECUTING`. In-flight copywriter work finishes; no further payment dispatches fire; campaign ends in `HALTED`.
4. **J4** — A copywriter fixture response contains a wallet-private-key-shaped hex string. The sanitizer replaces it with `[REDACTED:wallet-private-key]`; the planner's next prompt and the UI's payment ledger never contain the raw key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ad-spend-payment-agent demonstrating the
planner-executor × sales-marketing cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
planner-executor-sales-marketing-ad-spend-payment-agent. Java package
io.akka.samples.aiadsgenerationagentwithcryptopayments. Akka 3.6.0.
HTTP port 9826.

Components to wire (exactly):
- 3 AutonomousAgents:
  * CampaignPlannerAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PLAN_CAMPAIGN).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(DECIDE_NEXT).maxIterationsPerTask(3)),
      capability(TaskAcceptance.of(COMPOSE_REPORT).maxIterationsPerTask(2)).
    System prompt from prompts/campaign-planner.md. PLAN_CAMPAIGN returns
    CreativeLedger. DECIDE_NEXT returns a NextCampaignStep tagged union
    (Continue(DispatchDecision) | Replan(CreativeLedger) | Complete | Fail).
    COMPOSE_REPORT returns CampaignReport.
  * CopywriterAgent — capability(TaskAcceptance.of(WRITE_CREATIVE)
      .maxIterationsPerTask(2)).
    Prompt from prompts/copywriter.md. Returns CreativeResult.
  * PaymentExecutorAgent — capability(TaskAcceptance.of(EXECUTE_PAYMENT)
      .maxIterationsPerTask(2)).
    Prompt from prompts/payment-executor.md. Returns PaymentResult.

- 1 Workflow CampaignWorkflow with steps:
  planStep -> [loop entry] checkHaltStep -> proposeStep -> guardrailStep ->
  approvalGateStep (PAYMENT only) -> dispatchStep -> sanitizeStep ->
  recordStep -> decideStep
  -> [back to checkHaltStep or to composeReportStep / failStep / haltedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45),
    approvalGateStep ofSeconds(600) (waits up to 10 min for human),
    dispatchStep ofSeconds(120), decideStep ofSeconds(45),
    composeReportStep ofSeconds(60).
    defaultStepRecovery(maxRetries(2).failoverTo(CampaignWorkflow::error)).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions
  to haltedStep (emits CampaignHaltedOperator on CampaignEntity).
  guardrailStep runs PaymentGuardrail.vet(DispatchDecision, remainingBudgetWei);
  on reject records a CreativeBlocked entry and loops back to proposeStep.
  approvalGateStep is PAYMENT path only: writes ApprovalEntity and polls
  until ApprovalGranted or ApprovalRejected; on rejected loops to proposeStep.
  dispatchStep uses switch on DispatchDecision.executor to call
  CopywriterAgent (WRITE_CREATIVE) or PaymentExecutorAgent (EXECUTE_PAYMENT).
  sanitizeStep applies CredentialScrubber.scrub to CreativeResult.content
  or PaymentResult failure messages.
  recordStep calls CampaignEntity.recordCreative(entry) or
  CampaignEntity.recordPayment(entry).
  decideStep calls forAutonomousAgent(CampaignPlannerAgent.class, DECIDE_NEXT);
  on Continue or Replan loops; on Complete transitions to composeReportStep
  -> completeStep; on Fail transitions to failStep.

- 1 EventSourcedEntity CampaignEntity holding Campaign state.
  emptyState() returns Campaign.initial("", null) with no commandContext().
  Commands: createCampaign, recordPlan, recordDispatch, recordBlock,
  recordCreative, requestPaymentApproval, recordPaymentApproved,
  recordPaymentRejected, recordPayment, reviseLedger, completeCampaign,
  failCampaign, haltOperator, timeoutFail, getCampaign.
  Events as listed in SPEC §5.

- 1 EventSourcedEntity ApprovalEntity keyed by (campaignId + ":" + placementId).
  State ApprovalRequest. Commands: requestApproval, grant, reject, get.
  Events: ApprovalRequested, ApprovalGranted, ApprovalRejected.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get.
  Events: HaltRequested, HaltCleared.

- 1 EventSourcedEntity CampaignQueue with command enqueueBrief(campaignId,
  title, budgetWei, requestedBy) emitting BriefSubmitted.

- 1 View CampaignView with row type CampaignRow (mirror of Campaign minus heavy
  ledger payloads — truncate to last 3 creative entries plus counts; last 3
  placement entries; the UI fetches the full campaign by id on click).
  ONE query getAllCampaigns SELECT * AS campaigns FROM campaign_view.
  No WHERE status filter (Lesson 2).

- 1 Consumer CampaignRequestConsumer subscribed to CampaignQueue events;
  on BriefSubmitted starts a CampaignWorkflow with campaignId as the
  workflow id.

- 2 TimedActions:
  * BriefSimulator — every 90s, reads next line from
    src/main/resources/sample-events/brief-prompts.jsonl and calls
    CampaignQueue.enqueueBrief.
  * StaleCampaignMonitor — every 60s, queries CampaignView.getAllCampaigns,
    filters EXECUTING and AWAITING_APPROVAL campaigns older than 10 minutes,
    calls CampaignEntity.timeoutFail.

- 2 HttpEndpoints:
  * CampaignEndpoint at /api with POST /campaigns, GET /campaigns,
    GET /campaigns/{id}, GET /campaigns/sse,
    GET /campaigns/{id}/approvals,
    POST /campaigns/{id}/approvals/{placementId}/approve,
    POST /campaigns/{id}/approvals/{placementId}/reject,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving YAML/MD from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- PlannerTasks.java declaring three Task<R> constants: PLAN_CAMPAIGN
  (resultConformsTo CreativeLedger), DECIDE_NEXT (NextCampaignStep),
  COMPOSE_REPORT (CampaignReport).
- ExecutorTasks.java declaring two Task<R> constants: WRITE_CREATIVE
  (resultConformsTo CreativeResult), EXECUTE_PAYMENT (resultConformsTo
  PaymentResult).
- Domain records as listed in SPEC §5, plus a NextCampaignStep sealed
  interface with permits Continue(DispatchDecision), Replan(CreativeLedger),
  Complete(CampaignReport stub), Fail(String reason).
- application/CredentialScrubber.java — deterministic regex/entropy scrubber.
  Patterns: Ethereum private key hex (64 hex chars after 0x or bare),
  BIP-39-shaped word sequences (12 or 24 space-delimited lowercase words
  each 3–8 chars), generic bearer tokens, JWTs, high-entropy strings ≥ 32
  chars with Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:wallet-private-key], [REDACTED:seed-phrase],
  [REDACTED:bearer-token], [REDACTED:jwt], [REDACTED:high-entropy].
- application/PaymentGuardrail.java — deterministic vetter for payment
  dispatches. Reject if (a) proposedSpendWei > remainingBudgetWei,
  (b) destinationWallet is not in the allow-list loaded from
  sample-data/wallet-allowlist.json, (c) token is not ETH or USDC.
  Non-payment dispatches (COPYWRITER) pass through unconditionally.
- src/main/resources/application.conf with
  akka.javasdk.dev-mode.http-port = 9826 and akka.javasdk.agent
  model-provider blocks for anthropic (claude-sonnet-4-6), openai (gpt-4o),
  googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/brief-prompts.jsonl with 8 canned
  campaign briefs (varying brand voice, audience, ad format, and budget).
- src/main/resources/sample-data/wallet-allowlist.json — 5 pre-approved
  destination wallets for the placement allow-list.
- src/main/resources/sample-data/creative-fixtures.jsonl — 8 canned ad copy
  drafts used by CopywriterAgent mock. One entry must contain a bare 64-char
  hex string for the J4 sanitizer test.
- src/main/resources/sample-data/payment-fixtures.jsonl — 6 canned payment
  results (tx hashes, one failure) used by PaymentExecutorAgent mock.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint).
- eval-matrix.yaml at project root with 3 controls (G1, HI1, HO1) and
  a matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at project root pre-filling sector, decisions, data,
  capabilities, model_family, oversight from sales-marketing domain;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/campaign-planner.md, prompts/copywriter.md,
  prompts/payment-executor.md loaded at agent startup as system prompts.
- README.md at project root: title "Akka Sample: AI Ads Generation Agent
  with Crypto Payments", one-line pitch, prerequisites (integration form
  host-software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license.
  NO Configuration section. NO governance-mechanisms section.
  NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs acceptable. Five tabs matching the
  formal exemplar: Overview, Architecture (4 mermaid diagrams + click-to-
  expand component table with syntax-highlighted Java snippets), Risk Survey
  (7 sub-tabs from governance.html; unanswered .qb opacity 0.45), Eval
  Matrix (5-column table with click-to-expand rows), App UI (brief-submission
  form + pending-approvals pane + operator halt/resume control + live list
  with status pills and expand-on-click for ledgers and report).
  Browser title exactly:
  <title>Akka Sample: AI Ads Generation Agent with Crypto Payments</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and proceed
  silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml.
    (e) Type once in this session — value in Claude session memory only.
- NEVER write the key value to any file. No .env, no entry in
  application.conf, no secrets.yaml. Akka records only the REFERENCE.
- Generated Bootstrap.java fails fast with a clear message if the key
  reference does not resolve at runtime (never echoes key material).

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java dispatching on agent class name and
  Task<R> id. Each branch reads from
  src/main/resources/mock-responses/<agent-name>.json:
    campaign-planner.json — keyed by task id:
      "PLAN_CAMPAIGN" → 5 CreativeLedger entries spanning both COPYWRITER
      and PAYMENT steps with plausible budget splits.
      "DECIDE_NEXT" → 6 NextCampaignStep entries covering Continue
      (alternating COPYWRITER and PAYMENT dispatches), Replan, Complete, Fail.
      "COMPOSE_REPORT" → 4 CampaignReport entries with 60–120 word summaries,
      placement lists, and totalSpentWei values.
    copywriter.json — 6 CreativeResult entries; one entry's content must
      include a bare 64-char hex string (for the J4 sanitizer test).
    payment-executor.json — 5 PaymentResult entries (4 ok=true with tx hashes,
      1 ok=false with failureReason) mapped to the allow-listed wallets.
- MockModelProvider.seedFor(campaignId) makes selection deterministic per
  campaign id.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on planStep,
  proposeStep, approvalGateStep, dispatchStep, decideStep, composeReportStep.
- Lesson 6: Optional<T> for every nullable field on CampaignView row,
  Campaign entity state, and ApprovalRequest.
- Lesson 7: PlannerTasks.java and ExecutorTasks.java declaring all Task<R>
  constants.
- Lesson 8: model-name values: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build".
- Lesson 10: HTTP port 9826 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box".
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides
  AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute only.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
