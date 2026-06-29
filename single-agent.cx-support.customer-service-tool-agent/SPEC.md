# SPEC — customer-service-tool-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** CXAgent.
**One-line pitch:** A customer sends a support message; one AI agent calls the right tools — order lookup, account info, refund submission, inventory check — and returns a grounded reply. Three governance mechanisms gate every tool call and every customer-facing sentence.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the customer-experience (CX) support domain. One `SupportAgent` (AutonomousAgent) manages the entire tool-use loop; the surrounding components only prepare its context, gate its tool calls, and audit its output. Three governance mechanisms are wired around the agent:

- A **PII sanitizer** strips customer identifiers (names, email addresses, phone numbers, payment-card numbers, address fragments) from the agent's structured log output before it reaches any downstream system — the raw values are preserved on the entity for audit only.
- A **before-tool-call guardrail** (`ToolCallGuardrail`) fires before every tool invocation. It blocks any refund or account-write call whose amount or scope exceeds a configurable auto-approve threshold; those calls return a structured rejection that tells the agent to escalate instead. Read-only lookups pass through immediately.
- A **before-agent-response guardrail** (`ReplyGuardrail`) validates the agent's candidate reply on every turn. It checks that no raw PAN, SSN, or full-account-number appears in the reply text, that the reply references at least one tool result (grounding check), and that the reply is within the character limit. A failing reply triggers a retry inside the same task.

The blueprint shows that a single-agent tool-use loop does not mean ungoverned — two independent guardrails sit on the in-loop tool calls and the outbound reply, and a sanitizer covers the log path.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks or types a **customer identifier** (e.g., `CUST-1042`) from a dropdown pre-populated with seeded customers.
2. The user types a **support message** in the message textarea (or picks one of four seeded scenarios: "Where is my order?", "I want a refund for order ORD-5501", "Is the Apex Pro headset still in stock?", "Please update my email address").
3. The user clicks **Send**. The UI POSTs to `/api/conversations` (new session) or `/api/conversations/{id}/messages` (continuing session) and receives a `conversationId`.
4. The conversation card appears in the live list in `OPEN` state. Within ~1 s it transitions to `ACTIVE` — the agent is processing.
5. Within ~10–30 s the agent finishes. The card transitions to `REPLIED`. The reply appears in the conversation thread with a tool-calls summary showing which tools fired and what each returned.
6. If the before-tool-call guardrail blocked a tool call, the conversation card shows a `GUARDRAIL_BLOCKED` badge on the affected tool call row. The agent's final reply explains the block (e.g., "Your refund of $420 exceeds the auto-approve limit — I've opened escalation ticket ESC-9910 on your behalf").
7. The user can send a follow-up message in the same session; the agent receives the full conversation history in its task context.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SupportEndpoint` | `HttpEndpoint` | `/api/conversations/*` — new session, send message, list, get, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-session lifecycle: open → active → replied → resolved / escalated / failed. Source of truth. | `SupportEndpoint`, `ConversationWorkflow` | `ConversationView` |
| `ConversationWorkflow` | `Workflow` | One workflow per conversation session. Steps: `agentStep` → `screenStep`. On escalation trigger: `escalateStep`. | started by `SupportEndpoint` on first message | `SupportAgent`, `ConversationEntity` |
| `SupportAgent` | `AutonomousAgent` | The one decision-making LLM. Receives conversation history + available tools; calls tools in a loop; returns `AgentReply`. Before-tool-call and before-agent-response guardrails are registered on this agent. | invoked by `ConversationWorkflow` | returns `AgentReply` |
| `ToolCallGuardrail` | supporting class | Implements the before-tool-call hook. Blocks write-tool calls above threshold; allows read-only calls. | wired on `SupportAgent` | returns allow/block to agent loop |
| `ReplyGuardrail` | supporting class | Implements the before-agent-response hook. Checks PAN/SSN absence, grounding, character limit. | wired on `SupportAgent` | returns allow/reject to agent loop |
| `ReplyScreener` | `Consumer` | Subscribes to `AgentReplied` events; runs PII sanitizer over the reply log payload; emits `ReplyScreened`. | `ConversationEntity` events | `ConversationEntity` |
| `ConversationView` | `View` | Read model: one row per session for the UI. | `ConversationEntity` events | `SupportEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record CustomerMessage(
    String messageId,
    String customerId,
    String text,
    Instant sentAt
) {}

record ToolCall(
    String toolName,
    Map<String, String> arguments,
    String rawResult,
    ToolCallOutcome outcome,       // ALLOWED, BLOCKED
    Optional<String> blockReason
) {}

enum ToolCallOutcome { ALLOWED, BLOCKED }

record AgentReply(
    String replyText,
    List<ToolCall> toolCalls,
    ReplyOutcome outcome,          // SENT, BLOCKED_AND_RETRIED
    Instant repliedAt
) {}

enum ReplyOutcome { SENT, BLOCKED_AND_RETRIED }

record ScreenedPayload(
    String redactedReplyText,
    List<String> piiCategoriesFound
) {}

record ConversationTurn(
    CustomerMessage message,
    Optional<AgentReply> reply
) {}

record Conversation(
    String conversationId,
    String customerId,
    List<ConversationTurn> turns,
    Optional<ScreenedPayload> lastScreened,
    ConversationStatus status,
    Instant openedAt,
    Optional<Instant> closedAt
) {}

enum ConversationStatus {
    OPEN, ACTIVE, REPLIED, ESCALATED, RESOLVED, FAILED
}
```

Events on `ConversationEntity`: `ConversationOpened`, `MessageReceived`, `AgentProcessingStarted`, `AgentReplied`, `ReplyScreened`, `ConversationEscalated`, `ConversationResolved`, `ConversationFailed`.

Every nullable lifecycle field on the `Conversation` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ customerId, messageText }` → `{ conversationId }`.
- `POST /api/conversations/{id}/messages` — body `{ messageText }` → `{ messageId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation with full turn history.
- `GET /api/conversations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: CXAgent</title>`.

The App UI tab is a two-column layout: a left rail with the live list of conversation sessions (status pill + age + customer id) and a right pane with the selected session's conversation thread — each turn shows the customer message, the tool-calls summary (tool name, outcome badge, truncated result), and the agent reply.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `ReplyScreener` Consumer): redacts customer names, email addresses, phone numbers, payment-card numbers, postal addresses, and account-id-like tokens from the agent reply log payload before the event is emitted downstream. The raw reply is preserved on the entity for audit; only the screened form is shown in the UI.
- **G1 — before-tool-call guardrail** (`ToolCallGuardrail`): fires before every tool invocation on `SupportAgent`. Read-only tools (`lookupOrder`, `getAccountInfo`, `checkInventory`) always pass through. Write tools (`processRefund`, `updateAccount`) are allowed only when the operation amount or scope is within the auto-approve policy; calls that exceed it return a structured block with `blockReason` so the agent can draft an escalation reply instead. The block is recorded as `ToolCallOutcome.BLOCKED` on the `ToolCall` record.
- **G2 — before-agent-response guardrail** (`ReplyGuardrail`): runs on every turn of `SupportAgent`. Asserts the candidate reply contains no raw PAN (16-digit card number pattern), no raw SSN pattern, no raw account number, that the reply references at least one tool result (grounding check), and that reply length is within 1000 characters. On failure returns a structured rejection naming the failed check; the agent loop consumes one of its 4 iterations and retries. Passing replies flow through to `screenStep`.

## 9. Agent prompts

- `SupportAgent` → `prompts/support-agent.md`. The single decision-making LLM. System prompt instructs it on available tools, escalation policy, reply style, and PII hygiene (never repeat raw card numbers or government IDs in the reply).

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Customer asks for order status → agent calls `lookupOrder` → reply names the order status, ETA, and carrier. Appears in REPLIED state within 30 s.
2. **J2** — Customer requests a refund of $420 (above the $250 auto-approve threshold) → `before-tool-call` guardrail blocks `processRefund` → agent escalates → card shows `ESCALATED` status and the escalation ticket id.
3. **J3** — Agent's first reply draft contains a credit-card number pattern → `before-agent-response` guardrail rejects it → retry produces a clean reply → UI shows `REPLIED` (never the blocked draft).
4. **J4** — Service log for any session shows only redacted PII tokens (`[REDACTED-EMAIL]`, `[REDACTED-PCN]`), never raw values; the entity's `lastScreened.piiCategoriesFound` lists what was redacted.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system.

```
Create a sample named cx-agent demonstrating the single-agent × cx-support cell. Runs out of
the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-customer-service-tool-agent. Java package
io.akka.samples.customerserviceagentwithtools. Akka 3.6.0. HTTP port 9930.

Components to wire (exactly):

- 1 AutonomousAgent SupportAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/support-agent.md>) and
  .capability(TaskAcceptance.of(HANDLE_SUPPORT_QUERY).maxIterationsPerTask(4))
  .tools(OrderTools.class, AccountTools.class, InventoryTools.class, EscalationTools.class).
  The task receives the conversation history and customer context as its instruction text
  (formatted via formatHistory(turns)); no attachment is used — history is short enough
  to inline. Output: AgentReply{replyText: String, toolCalls: List<ToolCall>,
  outcome: ReplyOutcome, repliedAt: Instant}. The agent is configured with:
    (a) a before-tool-call guardrail (G1) registered via the agent's guardrail-configuration
        block, bound to ToolCallGuardrail.
    (b) a before-agent-response guardrail (G2) registered via the same block, bound to
        ReplyGuardrail.
  On a G2 rejection the agent loop retries the response within its 4-iteration budget.

- 1 Workflow ConversationWorkflow per conversationId. Steps:
  * agentStep — emits AgentProcessingStarted, then calls
    componentClient.forAutonomousAgent(SupportAgent.class, "agent-" + conversationId)
    .runSingleTask(TaskDef.instructions(formatHistory(conversation.turns)))
    — returns a taskId, then forTask(taskId).result(HANDLE_SUPPORT_QUERY) to fetch
    the AgentReply. On success calls ConversationEntity.recordReply(reply).
    WorkflowSettings.stepTimeout 90s with defaultStepRecovery maxRetries(2)
    .failoverTo(ConversationWorkflow::error).
  * screenStep — waits up to 10s for ReplyScreened event (polled via
    ConversationEntity.getConversation every 1s); once screened, transitions workflow
    to terminal success. WorkflowSettings.stepTimeout 15s.
  * escalateStep — called when G1 blocks a tool call and the agent returns an escalation
    reply; calls ConversationEntity.escalate(ticketId). WorkflowSettings.stepTimeout 10s.
  * error step — calls ConversationEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId: String, customerId: String, turns: List<ConversationTurn>,
  lastScreened: Optional<ScreenedPayload>, status: ConversationStatus,
  openedAt: Instant, closedAt: Optional<Instant>}. ConversationStatus enum: OPEN, ACTIVE,
  REPLIED, ESCALATED, RESOLVED, FAILED. Events: ConversationOpened{customerId},
  MessageReceived{message}, AgentProcessingStarted{}, AgentReplied{reply},
  ReplyScreened{screened}, ConversationEscalated{escalationTicketId},
  ConversationResolved{}, ConversationFailed{reason}. Commands: open, receiveMessage,
  markActive, recordReply, recordScreened, escalate, resolve, fail, getConversation.
  emptyState() returns Conversation.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer ReplyScreener subscribed to ConversationEntity events; on AgentReplied runs
  a regex + heuristic PII redaction pipeline (email, phone, payment-card number,
  postal address, person-name heuristic, account-id-like tokens) over replyText, computes
  the list of categories found, builds ScreenedPayload, then calls
  ConversationEntity.recordScreened(screened).

- 1 View ConversationView with row type ConversationRow (mirrors Conversation minus
  raw replyText — the audit log keeps the raw; the view holds the screened form for the UI).
  Table updater consumes ConversationEntity events. ONE query getAllConversations:
  SELECT * AS conversations FROM conversation_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SupportEndpoint at /api with
    POST /conversations (body {customerId, messageText}; mints conversationId;
      calls ConversationEntity.open + .receiveMessage; starts ConversationWorkflow;
      returns {conversationId}),
    POST /conversations/{id}/messages (body {messageText}; calls
      ConversationEntity.receiveMessage; restarts/continues workflow; returns {messageId}),
    GET /conversations (list from getAllConversations, newest-first),
    GET /conversations/{id} (one row with full turn history),
    GET /conversations/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving YAML/MD from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

- Tool classes (each is a plain Java class with @ComponentId, no AutonomousAgent extension):
  * OrderTools.java — lookupOrder(orderId: String): OrderRecord (seeded in-process store).
  * AccountTools.java — getAccountInfo(customerId: String): AccountRecord; updateAccount
    (customerId: String, newEmail: String): UpdateResult.
  * InventoryTools.java — checkInventory(productSku: String): InventoryRecord.
  * EscalationTools.java — openEscalation(customerId: String, reason: String, amount: double):
    EscalationTicket (mints ESC-{n} id, stores in-process).

  Seeded in-process data (src/main/resources/sample-data/):
    orders.jsonl — 6 orders across 3 seeded customers.
    accounts.jsonl — 3 customer accounts.
    products.jsonl — 5 product SKUs with stock levels.

Companion files:

- SupportTasks.java declaring one Task<R> constant: HANDLE_SUPPORT_QUERY = Task
  .name("Handle support query").description("Read the conversation history, call the
  appropriate tools, and return an AgentReply").resultConformsTo(AgentReply.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records CustomerMessage, ToolCall, ToolCallOutcome, AgentReply, ReplyOutcome,
  ScreenedPayload, ConversationTurn, Conversation, ConversationStatus. Also: OrderRecord,
  AccountRecord, InventoryRecord, EscalationTicket, UpdateResult.

- ToolCallGuardrail.java implementing the before-tool-call hook. Reads the tool name and
  arguments from the pending call; allows all read-only tools unconditionally; for
  processRefund checks arguments.get("amount") <= 250.0 (configurable via application.conf
  cx.refund.auto-approve-limit); for updateAccount checks arguments.containsKey("newEmail")
  only. Returns Guardrail.block(<structured-error>) for out-of-policy calls. Records outcome
  as ToolCallOutcome.BLOCKED on the ToolCall record.

- ReplyGuardrail.java implementing the before-agent-response hook. Checks: (1) no 13-16
  digit card-number pattern in replyText (regex), (2) no SSN pattern (###-##-#### or
  9 consecutive digits), (3) reply references at least one of the toolCalls by name
  (grounding), (4) replyText.length() <= 1000. On any failure returns
  Guardrail.reject(<structured-error>) naming the failed check.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9930, the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}, and cx.refund.auto-approve-limit = 250.0.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (S1, G1, G2) matching Section 8.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  decisions.authority_level = automated-action (the agent's tool calls are real writes,
  within the guardrail threshold), oversight.human_in_loop = false (within threshold)
  and oversight.human_on_loop = true (an operator reviews escalated sessions),
  failure.failure_modes including "hallucinated-tool-result", "pii-in-reply",
  "over-threshold-write-approved-by-mistake", "ungrounded-reply"; deployer fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/support-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Customer Service Agent with Tools",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of conversation session cards; right = selected session's conversation
  thread with per-turn message, tool-calls summary, and agent reply).
  Browser title exactly: <title>Akka Sample: CXAgent</title>. No subtitle on Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. The HANDLE_SUPPORT_QUERY branch reads
  src/main/resources/mock-responses/handle-support-query.json, picks one entry
  pseudo-randomly per call (seedFor(conversationId + turnIndex)), and deserialises into
  AgentReply.
- Per-task mock-response shapes for THIS blueprint:
    handle-support-query.json — 8 AgentReply entries covering all 4 seeded scenarios.
      Each entry has a replyText paragraph, a toolCalls list with at least one
      ToolCallOutcome.ALLOWED entry and realistic rawResult payloads. Include:
        - 2 entries for order-lookup queries (REPLIED, tools: lookupOrder)
        - 2 entries for refund queries below threshold (REPLIED, tools: processRefund ALLOWED)
        - 1 entry for refund query above threshold (ESCALATED, tools: processRefund BLOCKED)
        - 1 entry for inventory query (REPLIED, tools: checkInventory)
        - 1 deliberately malformed entry with a card-number pattern in replyText (triggers G2
          rejection + retry); select this on the FIRST iteration of every 4th conversation.
        - 1 entry for account-update query (REPLIED, tools: updateAccount ALLOWED)
- A MockModelProvider.seedFor(conversationId, turnIndex) helper makes selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SupportAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SupportTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (agentStep 90s, screenStep 15s,
  escalateStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the Conversation row record is Optional<T>.
- Lesson 7: SupportTasks.java with HANDLE_SUPPORT_QUERY = Task.name(...)
  .description(...).resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated model names.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9930 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (SupportAgent). The PII
  sanitizer (ReplyScreener Consumer) is rule-based regex — not an LLM — keeping the
  pattern's "one agent" promise honest.
- The before-tool-call guardrail fires before write-tool invocations; it is wired via the
  agent's guardrail-configuration block, not as a post-hoc check on the tool result.
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
