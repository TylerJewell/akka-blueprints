# SPEC — whatsapp-fintech-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** WhatsApp Fintech Agent.
**One-line pitch:** A customer sends a financial query or payment instruction over a WhatsApp-channel gateway; one AI agent resolves the request against the customer's account, calls the appropriate tools, and returns a structured response — while a PII sanitizer, a before-tool-call guardrail, and a HITL gate keep every funds-movement step auditable and human-confirmed above a value threshold.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the finance-analysis domain. One `FintechQueryAgent` (AutonomousAgent) carries the entire decision; the surrounding components prepare its input, guard its tool calls, and route large transactions to a human confirmation step. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw inbound message and the agent call — so the model never sees phone numbers, account numbers, or personal identifiers in the raw message text.
- A **before-tool-call guardrail** intercepts every tool call the agent wants to make. It validates that payment destination accounts exist in the seeded registry, that amounts are positive numbers within single-transaction limits, and that the calling message session is in an allowed state. A failing check returns a structured rejection to the agent loop so the task retries with a corrected tool call.
- An **application-level HITL gate** pauses the workflow when a payment instruction exceeds the configured threshold (default $500). The workflow suspends in `AWAITING_APPROVAL` state and waits for an external operator approval command before resuming execution. Below-threshold transactions skip the gate and execute immediately.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — three independent checks sit on either side of the one decision-making LLM call.

## 3. User-facing flows

The user opens the App UI tab, which simulates the operator console for the WhatsApp channel.

1. The operator pastes a raw customer message into the **Inbound message** textarea (or picks one of three seeded examples — an account balance query, a small fund transfer, a large fund transfer above the HITL threshold).
2. The operator picks a **customer account** from a dropdown (the seeded account fixtures) and clicks **Send message**. The UI POSTs to `/api/messages` and receives a `messageId`.
3. The card appears in the live list in `RECEIVED` state. Within ~1 s, it transitions to `SANITIZED` — the redacted message is visible in the card detail, with a small list of PII categories the sanitizer found.
4. Within ~5–30 s, the workflow's `respondStep` completes. The card transitions to `RESPONDING` then `RESPONSE_READY`. The agent response appears: a short reply text and a structured `AgentResponse` (intent classification, tool calls made, response text).
5. For below-threshold payments, the card immediately transitions to `EXECUTED`. For above-threshold payments, the card transitions to `AWAITING_APPROVAL`. The right pane shows the pending transaction detail and an **Approve** / **Reject** button for the operator.
6. When the operator clicks **Approve**, the workflow resumes and the card transitions to `EXECUTED`. Clicking **Reject** transitions to `HITL_REJECTED` and sends a denial response text.
7. The operator can submit another message; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MessageEndpoint` | `HttpEndpoint` | `/api/messages/*` — submit, list, get, SSE, approve/reject; serves `/api/metadata/*`. | — | `MessageEntity`, `MessageView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `MessageEntity` | `EventSourcedEntity` | Per-message lifecycle: received → sanitized → responding → response-ready → (awaiting-approval) → executed / hitl-rejected / failed. Source of truth. | `MessageEndpoint`, `MessageSanitizer`, `MessageWorkflow` | `MessageView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; redacts PII; calls `MessageEntity.attachSanitized`. | `MessageEntity` events | `MessageEntity` |
| `MessageWorkflow` | `Workflow` | One workflow per message. Steps: `awaitSanitizedStep` → `respondStep` → `hitlStep` (conditional) → `executeStep`. | started by `MessageSanitizer` once sanitized event lands | `FintechQueryAgent`, `MessageEntity` |
| `FintechQueryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives customer context and the sanitized message; calls balance/transaction tools; returns `AgentResponse`. | invoked by `MessageWorkflow` | returns response |
| `PaymentGuardrail` | supporting class | Before-tool-call hook registered on `FintechQueryAgent`. Validates destination accounts, amounts, and session state before any payment tool fires. | — | — |
| `HitlGate` | supporting class | Application-level HITL logic used by `hitlStep`. Evaluates whether a pending transaction exceeds the threshold and, if so, pauses the workflow until an operator decision arrives. | — | — |
| `MessageView` | `View` | Read model: one row per message for the UI. | `MessageEntity` events | `MessageEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record InboundMessage(
    String messageId,
    String customerId,
    String accountId,
    String rawText,
    String channelMessageId,
    Instant receivedAt
) {}

record SanitizedMessage(
    String redactedText,
    List<String> piiCategoriesFound
) {}

record ToolCall(
    String toolName,
    Map<String, String> args,
    String result
) {}

record AgentResponse(
    Intent intent,
    String responseText,
    List<ToolCall> toolCallsMade,
    Optional<PendingTransaction> pendingTransaction,
    Instant decidedAt
) {}
enum Intent { BALANCE_QUERY, TRANSACTION_HISTORY, FUND_TRANSFER, ACCOUNT_INFO, UNKNOWN }

record PendingTransaction(
    String fromAccountId,
    String toAccountId,
    BigDecimal amount,
    String currency,
    String description
) {}

record HitlDecision(
    String operatorId,
    HitlOutcome outcome,
    String note,
    Instant decidedAt
) {}
enum HitlOutcome { APPROVED, REJECTED }

record Message(
    String messageId,
    Optional<InboundMessage> inbound,
    Optional<SanitizedMessage> sanitized,
    Optional<AgentResponse> response,
    Optional<HitlDecision> hitlDecision,
    MessageStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MessageStatus {
    RECEIVED, SANITIZED, RESPONDING, RESPONSE_READY,
    AWAITING_APPROVAL, EXECUTED, HITL_REJECTED, FAILED
}
```

Events on `MessageEntity`: `MessageReceived`, `MessageSanitized`, `ResponseStarted`, `ResponseReady`, `ApprovalRequested`, `TransactionApproved`, `TransactionRejected`, `TransactionExecuted`, `MessageFailed`.

Every nullable lifecycle field on the `Message` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/messages` — body `{ customerId, accountId, rawText, channelMessageId }` → `{ messageId }`.
- `GET /api/messages` — list all messages, newest-first.
- `GET /api/messages/{id}` — one message.
- `GET /api/messages/sse` — Server-Sent Events; one event per state transition.
- `POST /api/messages/{id}/approve` — body `{ operatorId, note }` → `204`. Resumes the workflow from `AWAITING_APPROVAL`.
- `POST /api/messages/{id}/reject` — body `{ operatorId, note }` → `204`. Rejects and transitions to `HITL_REJECTED`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: WhatsApp Fintech Agent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted messages (status pill + intent badge + age) and a right pane with the selected message's detail — inbound text, sanitized text preview with PII category chips, agent response text, tool calls made table, and (when applicable) the pending transaction detail with Approve / Reject buttons.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): redacts phone numbers, account numbers, card-like tokens, national identifiers, person names, and postal addresses from the raw inbound message text before any LLM call. Records which categories were found.
- **G1 — before-tool-call guardrail**: runs on every tool call the `FintechQueryAgent` issues. Asserts that the destination `accountId` for any payment tool exists in the seeded account registry, that the `amount` is a positive number and does not exceed the single-transaction cap, and that the message session is in an allowed state (not already `EXECUTED` or `HITL_REJECTED`). On failure, returns a structured `invalid-tool-call` error to the agent loop so the task retries with a corrected tool call within its iteration budget.
- **H1 — HITL application gate**: runs inside `hitlStep` immediately after `ResponseReady` lands, when `response.pendingTransaction` is present and `amount >= large_value_threshold`. The workflow suspends in `AWAITING_APPROVAL`; an external `POST /api/messages/{id}/approve` or `/reject` resumes it. Below-threshold transactions skip this step entirely.

## 9. Agent prompts

- `FintechQueryAgent` → `prompts/fintech-query-agent.md`. The single decision-making LLM. System prompt instructs it to classify the customer's intent, call the appropriate tool(s), and return a typed `AgentResponse` with a short reply text and any pending transaction details.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer sends a balance query; within 15 s the agent response appears with the account balance and status `EXECUTED`.
2. **J2** — Customer sends a small fund transfer (below threshold); the guardrail validates the destination account; the transaction executes without a HITL pause; the card transitions to `EXECUTED`.
3. **J3** — Customer sends a large fund transfer (above threshold); the card enters `AWAITING_APPROVAL`; the operator clicks Approve; the card transitions to `EXECUTED`.
4. **J4** — Customer sends a large fund transfer; the operator clicks Reject; the card transitions to `HITL_REJECTED` and the customer receives a denial response.
5. **J5** — The agent issues a payment tool call with a destination account not in the registry; the `before-tool-call` guardrail rejects it; the agent retries with a corrected call; the final response is valid.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named whatsapp-fintech-agent demonstrating the single-agent × finance-analysis
cell. Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-finance-analysis-whatsapp-fintech-agent. Java package
io.akka.samples.whatsappfintechagent. Akka 3.6.0. HTTP port 9589.

Components to wire (exactly):

- 1 AutonomousAgent FintechQueryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/fintech-query-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_CUSTOMER_MESSAGE).maxIterationsPerTask(3)). The task
  receives the sanitized message text plus account context as the task instructions text.
  Output: AgentResponse{intent: Intent, responseText: String, toolCallsMade: List<ToolCall>,
  pendingTransaction: Optional<PendingTransaction>, decidedAt: Instant}. The agent is configured
  with a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries within its
  3-iteration budget.

- 1 Workflow MessageWorkflow per messageId with four steps:
  * awaitSanitizedStep — polls MessageEntity.getMessage every 1s; on message.sanitized().isPresent()
    advances to respondStep. WorkflowSettings.stepTimeout 15s.
  * respondStep — emits ResponseStarted, then calls componentClient.forAutonomousAgent(
    FintechQueryAgent.class, "agent-" + messageId).runSingleTask(
      TaskDef.instructions(formatContext(message))
    ) — returns a taskId, then forTask(taskId).result(HANDLE_CUSTOMER_MESSAGE) to fetch the
    AgentResponse. On success calls MessageEntity.recordResponse(response). WorkflowSettings
    .stepTimeout 60s with defaultStepRecovery maxRetries(2).failoverTo(MessageWorkflow::error).
  * hitlStep — inspects the recorded AgentResponse. If response.pendingTransaction().isPresent()
    AND amount >= large_value_threshold (read from config), emits ApprovalRequested and
    transitions MessageEntity to AWAITING_APPROVAL; otherwise emits TransactionExecuted and
    advances. WorkflowSettings.stepTimeout 0 (unbounded — workflow waits for the external
    approval/reject command via MessageEntity.approve / MessageEntity.reject). Resume is
    triggered by the workflow callback registered in approve/reject entity commands.
  * executeStep — emits TransactionExecuted (or TransactionRejected if hitl decided Rejected).
    WorkflowSettings.stepTimeout 10s. error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity MessageEntity (one per messageId). State Message{messageId, inbound:
  Optional<InboundMessage>, sanitized: Optional<SanitizedMessage>, response:
  Optional<AgentResponse>, hitlDecision: Optional<HitlDecision>, status: MessageStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. MessageStatus enum: RECEIVED, SANITIZED,
  RESPONDING, RESPONSE_READY, AWAITING_APPROVAL, EXECUTED, HITL_REJECTED, FAILED. Events:
  MessageReceived{inbound}, MessageSanitized{sanitized}, ResponseStarted{}, ResponseReady{response},
  ApprovalRequested{pendingTransaction}, TransactionApproved{decision}, TransactionRejected{decision},
  TransactionExecuted{}, MessageFailed{reason}. Commands: receive, attachSanitized, markResponding,
  recordResponse, requestApproval, approve, reject, markExecuted, fail, getMessage. emptyState()
  returns Message.initial("") with no commandContext() reference (Lesson 3). Every Optional<T>
  field uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to MessageEntity events; on MessageReceived runs a
  regex+heuristic redaction pipeline (phone numbers, account numbers, card-like tokens,
  national IDs, postal addresses, person-name heuristic) over rawText, computes the list of
  categories found, builds SanitizedMessage, then calls MessageEntity.attachSanitized(sanitized).
  After attachSanitized lands, the same Consumer starts a MessageWorkflow with id =
  "msg-" + messageId.

- 1 View MessageView with row type MessageRow (mirrors Message minus inbound.rawText — the
  audit log keeps the raw; the view holds the sanitized form for the UI). Table updater consumes
  MessageEntity events. ONE query getAllMessages: SELECT * AS messages FROM message_view. No WHERE
  status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * MessageEndpoint at /api with POST /messages (body {customerId, accountId, rawText,
    channelMessageId}; mints messageId; calls MessageEntity.receive; returns {messageId}),
    GET /messages (list from getAllMessages, sorted newest-first), GET /messages/{id} (one row),
    GET /messages/sse (Server-Sent Events forwarded from the view's stream-updates), POST
    /messages/{id}/approve (body {operatorId, note}; calls MessageEntity.approve; 204),
    POST /messages/{id}/reject (body {operatorId, note}; calls MessageEntity.reject; 204),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- MessageTasks.java declaring one Task<R> constant: HANDLE_CUSTOMER_MESSAGE = Task
  .name("Handle customer message").description("Classify the customer intent, call the
  appropriate tools, and return an AgentResponse with a reply text").resultConformsTo(
  AgentResponse.class). DO NOT skip this — the AutonomousAgent requires its companion Tasks
  class (Lesson 7).

- Domain records InboundMessage, SanitizedMessage, ToolCall, AgentResponse, Intent,
  PendingTransaction, HitlDecision, HitlOutcome, Message, MessageStatus.

- PaymentGuardrail.java implementing the before-tool-call hook. On every tool call from the
  agent it (1) checks the destination accountId exists in the seeded account registry (loaded
  from classpath), (2) checks the amount is positive and does not exceed
  single_transaction_cap_usd (from config), (3) checks the message session is not already
  in a terminal status. On failure returns Guardrail.reject(<structured-error>).

- HitlGate.java — pure logic class used by hitlStep. Reads large_value_threshold_usd from
  config. Returns a HitlGateDecision{required: boolean, amount: BigDecimal, threshold:
  BigDecimal}.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9589 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. Also: large_value_threshold_usd = 500,
  single_transaction_cap_usd = 10000.

- src/main/resources/sample-events/accounts.jsonl with 5 seeded account fixtures
  (customerId, accountId, accountType, balanceUsd, currency, status).

- src/main/resources/sample-events/seed-messages.jsonl with 3 seeded inbound messages:
  a balance query (no PII risk, no payment), a small transfer request ($50, below threshold,
  contains a phone number), and a large transfer request ($1200, above threshold, contains an
  account number and phone number).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, H1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  data_classes.payment-card-data = TO_BE_COMPLETED_BY_DEPLOYER,
  decisions.authority_level = autonomous (the agent executes fund transfers; HITL applies
  only above threshold), oversight.human_in_loop = true (HITL gate), failure.failure_modes
  including "wrong-account-transfer", "pii-leakage-via-llm", "guardrail-bypass-attempt",
  "hitl-approval-delay", "intent-misclassification"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/fintech-query-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: WhatsApp Fintech Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of message cards; right = selected-message detail with inbound text, sanitized
  text preview, agent response, tool calls made, and HITL decision panel when applicable).
  Browser title exactly: <title>Akka Sample: WhatsApp Fintech Agent</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Reads src/main/resources/mock-responses/<task-id>.json,
  picks one entry pseudo-randomly per call (seedFor(messageId)).
- Per-task mock-response shapes for THIS blueprint:
    handle-customer-message.json — 6 AgentResponse entries covering all Intent values.
      Two balance queries (intent BALANCE_QUERY, no pendingTransaction), two small transfers
      (FUND_TRANSFER, amount < 500, valid toolCallsMade), two large transfers (FUND_TRANSFER,
      amount >= 500, valid toolCallsMade). Plus 1 MALFORMED entry with an invalid destination
      accountId — triggering the PaymentGuardrail rejection path. Mock selects the malformed
      entry on the FIRST iteration of every 4th message (modulo seed) so J5 is reproducible.
- MockModelProvider.seedFor(messageId) makes per-message selection deterministic.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. FintechQueryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. MessageTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (respondStep
  60s, awaitSanitizedStep 15s, executeStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the Message row record is Optional<T>.
- Lesson 7: MessageTasks.java with HANDLE_CUSTOMER_MESSAGE mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9589 declared in application.conf akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words.
- Lesson 24: generated static-resources/index.html includes mermaid CSS overrides and
  themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (FintechQueryAgent). HitlGate and
  PaymentGuardrail are supporting classes — neither is an LLM agent.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check after the agent returns.
- The Overview tab's Try-it card shows just "/akka:build".
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
