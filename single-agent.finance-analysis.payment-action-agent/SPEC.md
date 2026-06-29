# SPEC — antom-payment

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Antom Payment Action Agent.
**One-line pitch:** An operator sends a payment instruction (initiate, query, or refund); one AI agent decides which Antom API to call, validates authorization before every tool invocation, pauses for human confirmation on high-value amounts, and halts immediately when a fraud signal arrives.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `PaymentActionAgent` (AutonomousAgent) carries every API-dispatch decision; the surrounding components authorize, gate, and halt it. Three governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs before the agent touches any payment API — checking that the requested currency is in the permitted set, the method is allowed for the merchant category, and the instruction originates from an authorized source. A failing check returns a structured rejection so the agent can revise its plan rather than silently skipping the call.
- An **application-level HITL gate** interrupts the workflow when the payment amount exceeds a configurable threshold (default 10 000 USD-equivalent). The workflow pauses in `AWAITING_APPROVAL` until an operator posts an explicit confirmation; the agent does not invoke any tool during this wait.
- An **operator/regulator halt** fires when a `FraudSignalDetected` event lands on the payment entity during execution. The halt transitions the entity to `HALTED` and instructs the agent loop to stop all tool calls immediately, preserving the partial execution state for audit.

The blueprint shows that a single-agent pattern can carry meaningful financial-control surface without needing multiple agents.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills in a **Payment instruction** form: payment type (INITIATE / QUERY / REFUND), amount, currency, recipient identifier, payment method (CARD / BANK_TRANSFER / WALLET), and an optional memo.
2. The user clicks **Submit instruction**. The UI POSTs to `/api/payments` and receives a `paymentId`.
3. The card appears in the live list in `REQUESTED` state. Within ~1 s, it transitions to `AUTHORIZED` (guardrail passed) or `REJECTED` (guardrail blocked it — the reason is shown immediately).
4. If the amount exceeds the high-value threshold, the card transitions to `AWAITING_APPROVAL`. An **Approve / Deny** action appears on the right-pane detail. Other payments continue processing.
5. For approved or sub-threshold payments, the workflow's `executeStep` runs the `PaymentActionAgent`. Within ~10–30 s, the agent completes and the entity reaches `SETTLED` (for INITIATE / REFUND) or `QUERIED` (for QUERY). The right pane shows the Antom API response fields: transaction id, settled amount, fee, status code, timestamp.
6. If a fraud signal is detected at any point during `EXECUTING`, the card transitions to `HALTED`. The right pane shows the signal type and the last tool call that was in flight when the halt fired.
7. The user can submit another instruction; the live list keeps the history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `PaymentEndpoint` | `HttpEndpoint` | `/api/payments/*` — submit, list, get, approve, deny, SSE; serves `/api/metadata/*`. | — | `PaymentEntity`, `PaymentView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `PaymentEntity` | `EventSourcedEntity` | Per-payment lifecycle: requested → authorized/rejected → awaiting-approval → executing → settled/queried/halted. Source of truth. | `PaymentEndpoint`, `FraudSignalConsumer`, `PaymentWorkflow` | `PaymentView` |
| `FraudSignalConsumer` | `Consumer` | Subscribes to `FraudSignalDetected` events; calls `PaymentEntity.halt(signalType)`. | `PaymentEntity` events | `PaymentEntity` |
| `PaymentWorkflow` | `Workflow` | One workflow per payment. Steps: `authStep` → `hitlGateStep` (conditional) → `executeStep` → `recordStep`. | started by `PaymentEndpoint` after entity submit | `PaymentActionAgent`, `PaymentEntity` |
| `PaymentActionAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the authorized instruction as task instructions; invokes simulated Antom tools (`initiatePayment`, `queryPaymentStatus`, `initiateRefund`). Returns `PaymentResult`. | invoked by `PaymentWorkflow` | returns result |
| `AuthorizationGuardrail` | supporting class | Before-tool-call guardrail registered on `PaymentActionAgent`. | agent loop | passes/rejects |
| `PaymentView` | `View` | Read model: one row per payment for the UI. | `PaymentEntity` events | `PaymentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum PaymentType { INITIATE, QUERY, REFUND }
enum PaymentMethod { CARD, BANK_TRANSFER, WALLET }
enum PaymentStatus {
    REQUESTED, AUTHORIZED, REJECTED,
    AWAITING_APPROVAL, EXECUTING, SETTLED, QUERIED, HALTED, FAILED
}

record PaymentInstruction(
    String paymentId,
    PaymentType type,
    String currency,           // ISO 4217, e.g. "USD", "EUR", "CNY"
    long amountMinorUnits,     // amount in minor currency units (cents etc.)
    String recipientId,        // Antom merchant/payee identifier
    PaymentMethod method,
    String memo,               // optional human note
    String submittedBy,
    Instant submittedAt
) {}

record AntomApiResponse(
    String transactionId,
    String antomStatus,        // raw Antom status code, e.g. "SUCCESS", "PENDING"
    long settledAmountMinorUnits,
    long feeMinorUnits,
    String currency,
    Instant settledAt
) {}

record PaymentResult(
    String outcome,            // "settled" | "queried" | "refunded" | "error"
    AntomApiResponse apiResponse,
    String agentSummary,       // 1-2 sentences from the agent
    Instant decidedAt
) {}

record FraudSignal(
    String signalType,         // e.g. "velocity-breach", "blocked-recipient", "amount-anomaly"
    String detail,
    Instant detectedAt
) {}

record Payment(
    String paymentId,
    Optional<PaymentInstruction> instruction,
    Optional<PaymentResult> result,
    Optional<FraudSignal> fraudSignal,
    PaymentStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}
```

Events on `PaymentEntity`: `PaymentRequested`, `PaymentAuthorized`, `PaymentRejected`, `ApprovalRequired`, `ApprovalGranted`, `ApprovalDenied`, `ExecutionStarted`, `PaymentSettled`, `PaymentQueried`, `FraudSignalDetected`, `PaymentHalted`, `PaymentFailed`.

Every nullable lifecycle field on the `Payment` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/payments` — body `{ type, currency, amountMinorUnits, recipientId, method, memo, submittedBy }` → `{ paymentId }`.
- `GET /api/payments` — list all payments, newest-first.
- `GET /api/payments/{id}` — one payment.
- `POST /api/payments/{id}/approve` — operator approves a paused high-value payment.
- `POST /api/payments/{id}/deny` — operator denies a paused high-value payment.
- `GET /api/payments/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Antom Payment Action Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of payment instructions (status pill + payment type badge + amount + age) and a right pane with the selected payment's detail — instruction fields, Antom API response, agent summary, and any fraud signal or approval controls.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`guardrail`, `before-tool-call`): runs on every tool call the agent prepares. Asserts (1) the requested `currency` is in the deployer-configured permitted-currencies list, (2) the `method` is allowed for the instruction's merchant category, (3) the `recipientId` is not on the blocked-recipient registry, and (4) the instruction's `submittedBy` matches an authorized-operator identifier. On any failure returns a structured `authorization-failure` rejection so the agent can log the block and stop — it does not silently retry with a different tool.
- **H1 — application HITL gate** (`hitl`, `application`): `PaymentWorkflow.hitlGateStep` evaluates `instruction.amountMinorUnits` against the high-value threshold. If the amount exceeds the threshold, the workflow transitions the entity to `AWAITING_APPROVAL` and pauses via `workflow.pause()`. `PaymentEndpoint.approve` / `.deny` resume or terminate the workflow. The agent's `executeStep` does not start until the gate either passes or is bypassed for sub-threshold amounts.
- **H2 — operator/regulator halt** (`halt`, `operator-regulator-stop`): `FraudSignalConsumer` subscribes to `FraudSignalDetected` events. On receipt it calls `PaymentEntity.halt(fraudSignal)`, transitions the entity to `HALTED`, and instructs the agent loop to abort all pending tool calls. The halt is unconditional — it fires regardless of the current workflow step or iteration count.

## 9. Agent prompts

- `PaymentActionAgent` → `prompts/payment-action-agent.md`. The single decision-making LLM. System prompt instructs it to read the authorized payment instruction, select the appropriate Antom API tool, and return a `PaymentResult` with the API response, outcome, and a 1–2 sentence summary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Operator submits a sub-threshold INITIATE instruction in USD → within 30 s the payment reaches `SETTLED` with a non-null `transactionId` in the API response.
2. **J2** — Operator submits an instruction above the threshold → card pauses at `AWAITING_APPROVAL`; approving it unblocks the workflow; the card reaches `SETTLED`.
3. **J3** — A `FraudSignalDetected` event is injected mid-execution → the card transitions to `HALTED` immediately; no further tool calls are logged.
4. **J4** — An instruction with a blocked `recipientId` is submitted → the before-tool-call guardrail rejects the tool call; no Antom API is invoked; the card shows `REJECTED` with the guardrail reason.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named antom-payment demonstrating the single-agent × finance-analysis cell.
Runs out of the box (no external services — Antom API calls are simulated in-process).
Maven group io.akka.samples. Maven artifact single-agent-finance-analysis-payment-action-agent.
Java package io.akka.samples.antompayment. Akka 3.6.0. HTTP port 9678.

Components to wire (exactly):

- 1 AutonomousAgent PaymentActionAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/payment-action-agent.md>) and
  .capability(TaskAcceptance.of(EXECUTE_PAYMENT).maxIterationsPerTask(3)). The task receives
  the authorized PaymentInstruction as its instruction text. Output: PaymentResult{outcome,
  apiResponse: AntomApiResponse, agentSummary: String, decidedAt: Instant}. The agent is
  configured with a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via
  the agent's guardrail-configuration block. On guardrail rejection the agent loop receives
  a structured authorization-failure error and stops — it does NOT retry with the same tool.

- 1 Workflow PaymentWorkflow per paymentId with four steps:
  * authStep — runs AuthorizationGuardrail.check(instruction) in-process; on pass calls
    PaymentEntity.authorize(); on fail calls PaymentEntity.reject(reason). WorkflowSettings
    .stepTimeout 5s.
  * hitlGateStep — evaluates amountMinorUnits against HIGH_VALUE_THRESHOLD_MINOR_UNITS
    (default 1_000_000 = 10 000 USD in cents). If above threshold calls
    PaymentEntity.requireApproval() then pauses the workflow via workflow.pause() waiting
    for resume from PaymentEndpoint.approve / .deny. WorkflowSettings.stepTimeout 86400s
    (24 h operator window). If below threshold advances immediately to executeStep.
  * executeStep — emits ExecutionStarted, then calls
    componentClient.forAutonomousAgent(PaymentActionAgent.class, "agent-" + paymentId)
    .runSingleTask(TaskDef.instructions(formatInstruction(instruction))). Returns taskId,
    then forTask(taskId).result(EXECUTE_PAYMENT) to fetch PaymentResult. On success calls
    PaymentEntity.settle(result). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(PaymentWorkflow::error).
  * recordStep — calls PaymentEntity.recordResult(result). WorkflowSettings.stepTimeout 5s.
    error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity PaymentEntity (one per paymentId). State Payment{paymentId: String,
  instruction: Optional<PaymentInstruction>, result: Optional<PaymentResult>,
  fraudSignal: Optional<FraudSignal>, status: PaymentStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. PaymentStatus enum: REQUESTED, AUTHORIZED, REJECTED,
  AWAITING_APPROVAL, EXECUTING, SETTLED, QUERIED, HALTED, FAILED. Events: PaymentRequested,
  PaymentAuthorized, PaymentRejected, ApprovalRequired, ApprovalGranted, ApprovalDenied,
  ExecutionStarted, PaymentSettled, PaymentQueried, FraudSignalDetected, PaymentHalted,
  PaymentFailed. Commands: submit, authorize, reject, requireApproval, grantApproval,
  denyApproval, startExecution, settle, query, detectFraud, halt, fail, getPayment.
  emptyState() returns Payment.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside
  the event-applier.

- 1 Consumer FraudSignalConsumer subscribed to PaymentEntity events; on FraudSignalDetected
  calls PaymentEntity.halt(fraudSignal.signalType()). The halt is unconditional — it fires
  regardless of current workflow step.

- 1 View PaymentView with row type PaymentRow (mirrors Payment). Table updater consumes
  PaymentEntity events. ONE query getAllPayments: SELECT * AS payments FROM payment_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * PaymentEndpoint at /api with:
    POST /payments (body {type, currency, amountMinorUnits, recipientId, method, memo,
    submittedBy}; mints paymentId; calls PaymentEntity.submit; starts PaymentWorkflow;
    returns {paymentId}),
    GET /payments (list from getAllPayments, sorted newest-first),
    GET /payments/{id} (one row),
    POST /payments/{id}/approve (resumes paused workflow via workflow.resume(ApprovalGranted)),
    POST /payments/{id}/deny (resumes paused workflow via workflow.resume(ApprovalDenied)),
    GET /payments/sse (Server-Sent Events from view stream-updates),
    three /api/metadata/* endpoints serving YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- PaymentTasks.java declaring one Task<R> constant: EXECUTE_PAYMENT = Task.name("Execute
  payment").description("Invoke the appropriate Antom API for the authorized payment
  instruction and return a PaymentResult").resultConformsTo(PaymentResult.class). DO NOT
  skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records PaymentInstruction, AntomApiResponse, PaymentResult, FraudSignal, Payment,
  and enums PaymentType, PaymentMethod, PaymentStatus.

- AuthorizationGuardrail.java implementing the before-tool-call hook. Reads the planned
  tool call name and parameters from the agent loop, runs the four checks listed in
  eval-matrix.yaml G1, and either passes the call through or returns
  Guardrail.reject(<structured authorization-failure>) to block the call.

- AntomApiSimulator.java — deterministic in-process simulator for three Antom API calls:
  initiatePayment(instruction) → AntomApiResponse, queryPaymentStatus(transactionId) →
  AntomApiResponse, initiateRefund(transactionId, amount) → AntomApiResponse. The simulator
  uses a SHA-based seed on paymentId so the same instruction always produces the same
  simulated status code and transactionId. One in every 5 calls (by seed mod 5 == 0) returns
  a PENDING status to exercise the queried-not-settled path. One in every 7 calls (by seed
  mod 7 == 0) produces a FraudSignalDetected event (velocity-breach) so J3 is reproducible.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9678 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The PaymentActionAgent.definition()
  binds the configured provider.

- src/main/resources/sample-events/payment-instructions.jsonl with 6 seeded payment
  instructions: 2 INITIATE (one sub-threshold USD, one above-threshold EUR), 2 QUERY
  (referencing the seeded transaction ids), 1 REFUND, 1 with a blocked recipientId
  (to exercise the guardrail rejection path).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, H2) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with sector = finance, decisions.authority_level =
  autonomous-with-gate (agent executes autonomously but high-value payments gate on HITL),
  oversight.human_in_loop = true for high-value payments, data.data_classes.payment-card-data
  = true, failure.failure_modes including "unauthorized-payment", "fraud-signal-ignored",
  "amount-threshold-bypass", "blocked-recipient-reached"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/payment-action-agent.md loaded as the agent system prompt.

- README.md at the project root matching blueprint README.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of payment cards with status pill, type badge, amount, age; right = selected
  payment detail with instruction fields, Antom API response, agent summary, fraud signal or
  approval controls).
  Browser title exactly: <title>Akka Sample: Antom Payment Action Agent</title>.
  No subtitle on the Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(paymentId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    execute-payment.json — 6 PaymentResult entries covering outcome values
    (settled, queried, refunded, error). Each entry has a non-null AntomApiResponse with
    realistic transactionId, antomStatus, settledAmountMinorUnits, feeMinorUnits, currency,
    settledAt. Plus 1 entry that simulates a fraud-signal path (outcome = "halted") and
    1 entry with a deliberately mismatched currency so the before-tool-call guardrail
    has a test case.
- A MockModelProvider.seedFor(paymentId) helper makes per-payment selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. PaymentActionAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion PaymentTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (authStep 5s, hitlGateStep 86400s,
  executeStep 60s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Payment row record is Optional<T>.
- Lesson 7: PaymentTasks.java with EXECUTE_PAYMENT = Task.name(...).description(...)
  .resultConformsTo(PaymentResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9678 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk. Akka records only the
  reference (env-var name / file path / secrets URI).
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (PaymentActionAgent).
  The authorization guardrail (AuthorizationGuardrail) and the Antom simulator
  (AntomApiSimulator) are pure Java helpers — neither makes an LLM call.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check that runs after the agent returns.
- The HITL gate pauses the workflow via workflow.pause(); the agent's executeStep does not
  start until a resume signal arrives from PaymentEndpoint.approve or .deny.
- The halt mechanism is unconditional — FraudSignalConsumer calls halt regardless of workflow
  step, agent iteration count, or amount threshold.
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
