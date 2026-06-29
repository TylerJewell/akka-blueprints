# SPEC — react-chatbot

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** BasicToolCallingChatbot.
**One-line pitch:** A user sends a message; one AI agent calls in-process tools in a loop until it has enough information, then returns a final natural-language reply — every candidate reply blocked by a `before-agent-response` guardrail before it reaches the user.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `ChatAgent` (AutonomousAgent) owns the entire turn: it decides which tools to invoke, reads their results, and produces a `ChatReply`. The surrounding components manage conversation state, enforce the response policy, and surface the full history in the UI. One governance mechanism is wired:

- A **before-agent-response guardrail** (`ReplyGuardrail`) runs on every candidate reply before it leaves the agent loop. It checks for prohibited content categories (instructions to circumvent the system, personally-identifiable information volunteered by the agent itself, off-topic solicitations), returns a structured rejection on failure, and lets the agent loop retry within its iteration budget.

The blueprint shows that a tool-calling loop and a response guardrail are orthogonal: the agent can call as many tools as it needs without touching the guardrail; the guardrail fires exactly once per candidate final answer, not on intermediate tool-call steps.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a message into the **Message** textarea and optionally selects a **conversation** from the left rail (or starts a new one).
2. The user clicks **Send**. The UI POSTs to `/api/conversations/{id}/messages` and receives a `turnId`.
3. The conversation card in the left rail shows the turn as `PROCESSING`. The right pane shows the latest message bubble.
4. Within ~10–30 s, the agent completes its tool loop and produces a final reply. The `before-agent-response` guardrail accepts or rejects the candidate. If rejected, the agent retries; if all iterations are exhausted the turn lands in `FAILED`. On success, the card transitions to `COMPLETED` and the reply bubble appears below the user message.
5. The user can send another message in the same conversation; the history accumulates in the right pane.
6. Tool-call traces (tool name + result summary) are visible in a collapsible "Reasoning" section below each reply bubble.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create, list, get, send message, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle: holds history of turns, each turn going through PROCESSING → COMPLETED or FAILED. Source of truth. | `ChatEndpoint`, `ConversationWorkflow` | `ConversationView` |
| `ConversationWorkflow` | `Workflow` | One workflow per turn. Steps: `agentStep` → `recordStep`. Handles guardrail-rejected retries inside the agent loop; records the final reply. | started by `ChatEndpoint` after `TurnStarted` event | `ChatAgent`, `ConversationEntity` |
| `ChatAgent` | `AutonomousAgent` | The one decision-making LLM. Receives conversation history plus the current user message as task instructions; calls in-process tools in a loop; returns `ChatReply`. | invoked by `ConversationWorkflow` | returns reply |
| `ReplyGuardrail` | supporting class | `before-agent-response` hook on `ChatAgent`. Checks candidate replies for prohibited content, PII volunteered by the agent, and off-topic solicitations. Rejects with a structured code; accepts clean replies. | wired to `ChatAgent` | — |
| `ToolRegistry` | supporting class | In-process tool implementations called by `ChatAgent`: `search_knowledge_base`, `get_order_status`, `get_faq_answer`. | invoked by `ChatAgent` | returns tool results |
| `ConversationView` | `View` | Read model: one row per conversation for the UI list; one row per turn within each conversation. | `ConversationEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record UserMessage(
    String turnId,
    String content,
    String sentBy,
    Instant sentAt
) {}

record ToolCall(
    String toolName,
    String inputSummary,
    String resultSummary
) {}

record ChatReply(
    String content,
    List<ToolCall> toolCallTrace,
    Instant repliedAt
) {}

record Turn(
    String turnId,
    UserMessage message,
    Optional<ChatReply> reply,
    TurnStatus status,
    Optional<String> failureReason,
    Instant startedAt,
    Optional<Instant> finishedAt
) {}

record Conversation(
    String conversationId,
    String title,
    String createdBy,
    List<Turn> turns,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> lastActiveAt
) {}

enum TurnStatus { PROCESSING, COMPLETED, FAILED }
enum ConversationStatus { ACTIVE, CLOSED }
```

Events on `ConversationEntity`: `ConversationStarted`, `TurnStarted`, `ReplyRecorded`, `TurnFailed`, `ConversationClosed`.

Every nullable lifecycle field on `Turn` and `Conversation` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ title, createdBy }` → `{ conversationId }`.
- `POST /api/conversations/{id}/messages` — body `{ content, sentBy }` → `{ turnId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation with all turns.
- `GET /api/conversations/sse` — Server-Sent Events; one event per state transition across all conversations.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Basic Tool-Calling Chatbot</title>`.

The App UI tab is a two-column layout: a left rail with the live list of conversations (status pill + last-message preview + age) and a right pane with the selected conversation's full turn history — each turn shows the user message bubble, the collapsible tool-call trace, and the agent reply bubble.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every candidate reply turn of `ChatAgent`. Asserts the candidate reply does not contain: (1) system-prompt disclosure or instructions to ignore the assistant's configuration, (2) PII patterns that the agent volunteered rather than parroted from user input (e.g., fabricated phone numbers, emails), (3) content from categories listed in `guardrail-policy.json` (hate speech, self-harm, explicit content, financial advice beyond the scope claim). On failure, returns a structured `guardrail-reject` code (`policy-violation | pii-leak | scope-breach`) to the agent loop so the task retries within its 3-iteration budget. On guardrail exhaustion, the turn lands in `FAILED` and `ConversationEntity` emits `TurnFailed`.

## 9. Agent prompts

- `ChatAgent` → `prompts/chat-agent.md`. The single decision-making LLM. System prompt defines the assistant persona, the available tools (`search_knowledge_base`, `get_order_status`, `get_faq_answer`), the response-format contract (plain prose, no markdown headers in replies, tool calls as needed before the final answer), and the scope boundary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends a question that requires a `search_knowledge_base` tool call; within 30 s the reply appears with a non-empty tool-call trace and a coherent answer.
2. **J2** — The agent's first candidate reply on a turn is policy-violating (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a clean reply; the UI never displays the rejected content.
3. **J3** — A three-turn conversation retains all three user messages and three replies in the right pane history; the SSE stream delivers each transition within 1 s.
4. **J4** — Sending a message while a prior turn is still `PROCESSING` is rejected with `409 Conflict`; the UI shows the error inline without breaking the existing conversation state.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named basictoolcallingchatbot demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-react-chatbot. Java package io.akka.samples.basictoolcallingchatbot.
Akka 3.6.0. HTTP port 9524.

Components to wire (exactly):

- 1 AutonomousAgent ChatAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/chat-agent.md>) and
  .capability(TaskAcceptance.of(CHAT_TURN).maxIterationsPerTask(3))
  and tools registered via the ToolRegistry (search_knowledge_base, get_order_status,
  get_faq_answer). The agent is configured with a before-agent-response guardrail
  (see G1 in eval-matrix.yaml) registered via the agent's guardrail-configuration block.
  On guardrail rejection the agent loop retries the response within its 3-iteration budget.
  Output: ChatReply{content: String, toolCallTrace: List<ToolCall>, repliedAt: Instant}.

- 1 Workflow ConversationWorkflow per turnId with two steps:
  * agentStep — calls componentClient.forAutonomousAgent(ChatAgent.class,
    "chat-" + conversationId).runSingleTask(
      TaskDef.instructions(formatHistory(conversation.turns()) + "\nUser: " + message.content())
    ) — returns a taskId, then forTask(taskId).result(CHAT_TURN) to fetch the reply.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ConversationWorkflow::error).
  * recordStep — calls ConversationEntity.recordReply(reply). WorkflowSettings.stepTimeout 5s.
    error step calls ConversationEntity.failTurn(reason) and transitions to end.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State:
  Conversation{conversationId: String, title: String, createdBy: String,
  turns: List<Turn>, status: ConversationStatus, createdAt: Instant,
  lastActiveAt: Optional<Instant>}. ConversationStatus enum: ACTIVE, CLOSED.
  TurnStatus enum: PROCESSING, COMPLETED, FAILED. Events: ConversationStarted{title, createdBy},
  TurnStarted{message: UserMessage}, ReplyRecorded{turnId, reply: ChatReply},
  TurnFailed{turnId, reason: String}, ConversationClosed{}.
  Commands: startConversation, sendMessage, recordReply, failTurn, closeConversation,
  getConversation. emptyState() returns Conversation.initial("") with no commandContext()
  reference (Lesson 3). turns field starts as an empty list; each TurnStarted appends a new
  Turn; ReplyRecorded and TurnFailed update the matching turn by turnId.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation including
  all turns minus any internal orchestration fields). Table updater consumes ConversationEntity
  events. ONE query getAllConversations: SELECT * AS conversations FROM conversation_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * ChatEndpoint at /api with:
    POST /conversations (body {title, createdBy}; mints conversationId via UUID;
      calls ConversationEntity.startConversation; returns {conversationId}),
    POST /conversations/{id}/messages (body {content, sentBy}; mints turnId; calls
      ConversationEntity.sendMessage then starts ConversationWorkflow; returns {turnId};
      returns 409 if the conversation already has a PROCESSING turn),
    GET /conversations (list from getAllConversations, sorted newest-first),
    GET /conversations/{id} (one conversation with all turns),
    GET /conversations/sse (Server-Sent Events forwarded from the view's stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ChatTasks.java declaring one Task<R> constant: CHAT_TURN = Task.name("Chat turn")
  .description("Process the user message, call any needed tools, and return a ChatReply")
  .resultConformsTo(ChatReply.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records UserMessage, ToolCall, ChatReply, Turn, Conversation, TurnStatus,
  ConversationStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ChatReply.content, runs three checks (system-prompt disclosure pattern, PII fabrication
  patterns, scope-breach keyword list from guardrail-policy.json), and either passes the
  response through or returns Guardrail.reject(<structured-error-code>) to force the agent
  loop to retry.

- ToolRegistry.java — three in-process tool implementations, each returning realistic stub
  data loaded from src/main/resources/tools/<tool-name>.json:
  * search_knowledge_base(query: String) → List<KnowledgeArticle{title, snippet, url}>
  * get_order_status(orderId: String) → OrderStatus{orderId, status, estimatedDelivery}
  * get_faq_answer(topic: String) → FaqAnswer{topic, answer, relatedTopics: List<String>}

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9524 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. ChatAgent.definition() binds the
  configured provider via the per-agent override pattern from the akka-context docs.

- src/main/resources/tools/search_knowledge_base.json with 10 knowledge articles covering
  topics: shipping, returns, account management, product specs, billing, warranty.
- src/main/resources/tools/get_order_status.json with 5 order-status entries for order IDs
  ORD-001 through ORD-005, mixing DELIVERED, IN_TRANSIT, PROCESSING statuses.
- src/main/resources/tools/get_faq_answer.json with 8 FAQ entries covering common support
  topics.

- src/main/resources/guardrail-policy.json with the prohibited-content lists used by
  ReplyGuardrail: system-prompt-disclosure-patterns, pii-fabrication-patterns,
  scope-breach-keywords.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view block. No regulation_anchors.

- risk-survey.yaml at the project root with sector = cx-support, decisions.authority_level
  = inform-only (the agent's reply is informational; no automated action is taken),
  oversight.human_in_loop = false (the reply goes directly to the user), failure.failure_modes
  including "policy-violating-reply", "hallucinated-tool-result", "pii-leak-in-reply",
  "scope-breach", "guardrail-exhaustion"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/chat-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Basic Tool-Calling Chatbot",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of conversations; right = selected-conversation turn history with user bubbles,
  collapsible tool-call traces, and agent reply bubbles).
  Browser title exactly: <title>Akka Sample: Basic Tool-Calling Chatbot</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversationId + turnId)),
  and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    chat-turn.json — 8 ChatReply entries covering happy-path replies with tool-call traces
      (each entry has a non-empty content field and a toolCallTrace with 1–3 ToolCall entries).
      Plus 2 deliberately POLICY-VIOLATING entries (one with system-prompt disclosure language;
      one with a scope-breach keyword). The mock should select a violating entry on the FIRST
      iteration of every 3rd turn (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(conversationId, turnId) helper makes per-turn selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChatAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChatTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (agentStep
  60s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Turn and Conversation is Optional<T>.
- Lesson 7: ChatTasks.java with CHAT_TURN = Task.name(...).description(...)
  .resultConformsTo(ChatReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9524 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in narrative,
  marketing tone, competitor brand names.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements in the DOM.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ChatAgent). ToolRegistry
  provides in-process stubs — it is NOT an agent, NOT an LLM call, NOT a sub-agent. The
  guardrail check is NOT an LLM call.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
