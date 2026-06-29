# SPEC — mcp-chat-server

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ChatServer (MCP).
**One-line pitch:** A user sends a chat message; one AI agent reads the message, calls external tools via an MCP client, and streams a validated reply back — with a guardrail on every tool call and another on every outbound response.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the developer-tooling domain. One `ChatAgent` (AutonomousAgent) carries the entire decision; the surrounding components only route its input, audit its tool calls, and persist its output. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs immediately before the agent calls any MCP-registered tool — so dangerous or off-policy tool invocations are blocked before they reach an external service.
- A **before-agent-response guardrail** validates the agent's candidate reply on every turn: well-formed `ChatReply` JSON, reply text within the length limit, no disallowed content patterns. A rejected reply triggers a retry inside the same task iteration budget.

The blueprint shows that MCP-augmented agents require two distinct governance cut-points — one for outbound tool calls and one for inbound model responses — and that both can be registered on a single AutonomousAgent without a second LLM.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **conversation** from the left rail, or clicks **New conversation** to start one.
2. The user types a message into the **Message** textarea and clicks **Send**.
3. The UI POSTs to `/api/conversations/{id}/messages` and receives a `turnId`.
4. The message card appears in the conversation thread in `RECEIVED` state. Within 1 s the workflow starts; the card transitions to `RUNNING`.
5. The agent calls zero or more MCP tools. Each tool call passes through `ToolCallGuardrail`; blocked calls are counted and logged but do not abort the task.
6. Within ~10–30 s, the agent's reply is returned from the task. The `before-agent-response` guardrail validates it; if valid, the card transitions to `REPLIED` and the reply text appears.
7. If the agent's reply fails the content check, the agent retries (up to its iteration budget). If all iterations fail, the turn transitions to `FAILED` with a reason.
8. The user can send another message in the same conversation. The full turn history is visible in the thread.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create, list, get; `/api/conversations/{id}/messages` — send, SSE; `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation lifecycle and turn history. Source of truth. | `ChatEndpoint`, `ChatWorkflow` | `ConversationView` |
| `ChatWorkflow` | `Workflow` | One workflow per turn. Steps: `runAgentStep` → `recordReplyStep`. | started by `ChatEndpoint` after `MessageReceived` | `ChatAgent`, `ConversationEntity` |
| `ChatAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user message and conversation history as the task definition; calls MCP tools via the registered `MCPClient`; returns `ChatReply`. | invoked by `ChatWorkflow` | returns reply |
| `ToolCallGuardrail` | supporting class | `before-tool-call` hook registered on `ChatAgent`; validates every MCP tool invocation before it executes. | called by agent loop | pass-through or block |
| `ReplyGuardrail` | supporting class | `before-agent-response` hook registered on `ChatAgent`; validates the candidate reply before it leaves the agent loop. | called by agent loop | pass-through or reject |
| `ConversationView` | `View` | Read model: one row per conversation + embedded turn list. | `ConversationEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record UserMessage(
    String turnId,
    String conversationId,
    String text,
    String sentBy,
    Instant sentAt
) {}

record ToolCall(
    String toolName,
    String serverUrl,
    String inputSummary,    // truncated, for audit — not the full payload
    ToolCallOutcome outcome,
    Instant calledAt
) {}
enum ToolCallOutcome { ALLOWED, BLOCKED }

record ChatReply(
    String replyText,
    List<ToolCall> toolCallsAudited,
    int iterationsUsed,
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
enum TurnStatus { RECEIVED, RUNNING, REPLIED, FAILED }

record Conversation(
    String conversationId,
    String title,
    String createdBy,
    List<Turn> turns,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> lastActiveAt
) {}
enum ConversationStatus { ACTIVE, CLOSED }
```

Events on `ConversationEntity`: `ConversationCreated`, `MessageReceived`, `AgentRunStarted`, `ReplyRecorded`, `TurnFailed`, `ConversationClosed`.

Every nullable lifecycle field on `Turn` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ title, createdBy }` → `{ conversationId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation with full turn history.
- `POST /api/conversations/{id}/messages` — body `{ text, sentBy }` → `{ turnId }`.
- `GET /api/conversations/{id}/sse` — Server-Sent Events; one event per state transition on the conversation.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Chat Server (MCP)</title>`.

The App UI tab is a two-column layout: a left rail with the list of conversations (title, status pill, last-active age, turn count) and a right pane with the active conversation thread — user messages, agent replies, tool-call audit chips, and a message-entry box at the bottom.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs before every MCP tool invocation inside `ChatAgent`. Checks that the tool name is on the registered allow-list, that the server URL matches the configured MCP server (no SSRF-by-tool-call), and that the input payload does not exceed the configured size limit. Blocked calls are logged as `ToolCall{outcome: BLOCKED}` and returned to the agent as a structured refusal — the agent may choose a different tool or proceed without it. A blocked call does not abort the task.
- **G2 — before-agent-response guardrail**: runs on every turn of `ChatAgent`. Asserts the candidate response is well-formed `ChatReply` JSON, `replyText` length is within the configured limit, and `replyText` does not contain disallowed content patterns (configurable regex list). On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.

## 9. Agent prompts

- `ChatAgent` → `prompts/chat-agent.md`. The single decision-making LLM. System prompt instructs it to read the user message and conversation history, call MCP tools as needed, and return a `ChatReply` with a clear, grounded reply.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends a message in a new conversation; within 30 s the agent replies via one or more MCP tool calls and the reply appears in the thread.
2. **J2** — The agent attempts to call a tool not on the allow-list; the `before-tool-call` guardrail blocks it; the agent replies without using the blocked tool; the UI shows a blocked-tool chip on the turn card.
3. **J3** — The agent's first reply attempt fails the content check; the `before-agent-response` guardrail rejects it; the second iteration produces a valid reply; no malformed text ever reaches the UI.
4. **J4** — Two conversations run concurrently; each maintains independent turn histories and statuses.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mcp-chat-server demonstrating the single-agent × dev-code cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-mcp-chat-server. Java package io.akka.samples.chatservermcp. Akka 3.6.0.
HTTP port 9818.

Components to wire (exactly):

- 1 AutonomousAgent ChatAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/chat-agent.md>) and
  .capability(TaskAcceptance.of(CHAT_TURN).maxIterationsPerTask(4)). The agent is configured
  with BOTH guardrails: a before-tool-call hook (ToolCallGuardrail) and a
  before-agent-response hook (ReplyGuardrail) registered via the agent's
  guardrail-configuration block. The agent's MCP tools are injected via an MCPClient
  registered in application.conf under akka.javasdk.agent.mcp-servers (bundled mock server
  for the out-of-the-box path). Output: ChatReply{replyText: String,
  toolCallsAudited: List<ToolCall>, iterationsUsed: int, repliedAt: Instant}.

- 1 Workflow ChatWorkflow per turnId with two steps:
  * runAgentStep — emits AgentRunStarted on ConversationEntity, then calls
    componentClient.forAutonomousAgent(ChatAgent.class, "agent-" + conversationId)
    .runSingleTask(TaskDef.instructions(formatTurnContext(conversation, userMessage)))
    — returns a taskId, then forTask(taskId).result(CHAT_TURN) to fetch the reply.
    On success advances to recordReplyStep. WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(ChatWorkflow::error).
  * recordReplyStep — calls ConversationEntity.recordReply(reply). On success transitions
    to done. WorkflowSettings.stepTimeout 10s. error step calls
    ConversationEntity.failTurn(reason) and transitions to done.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId: String, title: String, createdBy: String, turns: List<Turn>,
  status: ConversationStatus, createdAt: Instant, lastActiveAt: Optional<Instant>}.
  ConversationStatus enum: ACTIVE, CLOSED.
  Turn{turnId: String, message: UserMessage, reply: Optional<ChatReply>, status: TurnStatus,
  failureReason: Optional<String>, startedAt: Instant, finishedAt: Optional<Instant>}.
  TurnStatus enum: RECEIVED, RUNNING, REPLIED, FAILED.
  Events: ConversationCreated{title, createdBy, createdAt},
  MessageReceived{message}, AgentRunStarted{turnId}, ReplyRecorded{turnId, reply},
  TurnFailed{turnId, reason}, ConversationClosed{}.
  Commands: create, receiveMessage, markRunning, recordReply, failTurn, close,
  getConversation. emptyState() returns Conversation.initial("") with empty turns list,
  status ACTIVE, no commandContext() reference (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside event-appliers.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation;
  includes embedded List<TurnRow> for the thread). Table updater consumes
  ConversationEntity events. TWO queries:
    getAllConversations: SELECT * AS conversations FROM conversation_view.
    getConversation: SELECT * FROM conversation_view WHERE conversationId = :conversationId.
  No WHERE status filter on the list query — Akka cannot auto-index enum columns (Lesson 2).

- 2 HttpEndpoints:
  * ChatEndpoint at /api with:
    POST /conversations (body {title, createdBy}; mints conversationId; calls
      ConversationEntity.create; returns {conversationId}),
    GET /conversations (list from getAllConversations, sorted newest-first),
    GET /conversations/{id} (one row from getConversation),
    POST /conversations/{id}/messages (body {text, sentBy}; mints turnId; calls
      ConversationEntity.receiveMessage; starts ChatWorkflow("turn-" + turnId); returns
      {turnId}),
    GET /conversations/{id}/sse (Server-Sent Events forwarded from the view's
      stream-updates for the specific conversationId),
    and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ChatTasks.java declaring one Task<R> constant: CHAT_TURN = Task.name("Chat turn")
  .description("Read the user message and conversation history, call MCP tools as needed,
  and return a ChatReply").resultConformsTo(ChatReply.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records UserMessage, ToolCall, ToolCallOutcome, ChatReply, Turn, TurnStatus,
  Conversation, ConversationStatus.

- ToolCallGuardrail.java implementing the before-tool-call hook. Checks:
  (1) tool name is on the allow-list read from application.conf,
  (2) server URL matches the configured MCP server base URL,
  (3) input payload size does not exceed configuredMaxToolInputBytes.
  On failure returns Guardrail.blockToolCall(<structured-reason>); the agent loop receives
  a structured refusal. A blocked call does NOT abort the task — the agent may choose an
  alternative. Appends a ToolCall{outcome: BLOCKED} entry to the audit list.

- ReplyGuardrail.java implementing the before-agent-response hook. Checks:
  (1) response parses into ChatReply,
  (2) replyText.length() <= configuredMaxReplyChars,
  (3) replyText does not match any pattern in configuredDisallowedPatterns (regex list from
  application.conf).
  On failure returns Guardrail.reject(<structured-error>); the agent loop retries within its
  4-iteration budget.

- MockMcpServer.java — a minimal in-process Starlette-compatible stub that listens on a
  fixed port (configured in application.conf as akka.javasdk.agent.mcp-servers[0].url).
  Exposes three mock tools: echo (returns input as-is), search (returns a canned list of
  results), time (returns the current ISO-8601 timestamp). Started as a managed lifecycle
  bean alongside the main service.

- src/main/resources/application.conf with:
  akka.javasdk.dev-mode.http-port = 9818
  akka.javasdk.agent.mcp-servers[0].url = "http://localhost:9900/mcp"
  akka.javasdk.agent.mcp-servers[0].allow-list = ["echo","search","time"]
  akka.javasdk.agent.mcp-servers[0].max-tool-input-bytes = 8192
  and the three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded conversation
  starters: a code debugging question, a repo search query, and a time-zone lookup. Each
  seed is a {title, createdBy, firstMessage} object for the UI's "Load seeded example"
  button.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, G2) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false (chat text is
  not classified as PII by default; deployer may override), decisions.authority_level =
  direct-action (the agent acts on tools directly, not advisory), oversight.human_in_loop =
  false (messages are automated responses), failure.failure_modes including
  "tool-call-ssrf", "disallowed-content-in-reply", "tool-output-injection",
  "iteration-budget-exhausted"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/chat-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Chat Server (MCP)", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  conversation list + new-conversation button; right = active conversation thread with
  message entry box at the bottom, tool-call audit chips on each agent turn, and turn
  status pills). Browser title exactly: <title>Akka Sample: Chat Server (MCP)</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct outputs per agent (see Mock LLM provider block below). Sets
        model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf; /akka:build
        forwards the value from the Claude session env to the JVM via the MCP tool's
        environment parameter.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml;
        /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time using the
        matching CLI (op / aws / vault).
    (e) Type once in this session — value lives only in Claude session memory; passed to the
        JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversationId +
  turnId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    chat-turn.json — 6 ChatReply entries covering realistic dev-tool responses: one echoing
    back the user input, one returning search results, one returning the current time, one
    a multi-tool response, one citing a blocked tool (the replyText explains the tool was
    unavailable and offers an alternative). Plus 2 deliberately MALFORMED entries (one
    whose replyText exceeds maxReplyChars by 1; one that is not valid ChatReply JSON) — the
    ReplyGuardrail blocks both, exercising the retry path.
- MockModelProvider.seedFor(conversationId, turnId) makes selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChatAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChatTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (runAgentStep 60s,
  recordReplyStep 10s, error 5s).
- Lesson 6: every Optional<T> lifecycle field uses Optional.empty() in initial state and
  Optional.of(...) inside event-appliers.
- Lesson 7: ChatTasks.java with CHAT_TURN = Task.name(...).description(...)
  .resultConformsTo(ChatReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9818 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in prose.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ChatAgent). Both
  guardrails are registered on that one agent — they do not require a second LLM.
- The before-tool-call guardrail MUST be registered as a hook on the agent, not as an
  external wrapper around the MCP client. The hook is the authoritative cut-point.
- A blocked tool call does NOT abort the task. The agent receives the structured refusal
  and may continue with alternative tools or without the blocked tool.
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
