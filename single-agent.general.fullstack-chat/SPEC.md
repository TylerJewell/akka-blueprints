# SPEC — gemini-fullstack

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** GeminiFullstackChat.
**One-line pitch:** A user opens a chat UI, types a message, and one AI agent generates a reply — the full conversation history persists durably in an EventSourcedEntity and streams to the browser over SSE without any polling.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `ChatAgent` (AutonomousAgent) carries every reply decision; the surrounding components only prepare its input, persist its output, and stream results to the frontend. The frontend is a self-contained HTML page served from the same Akka process — no separate build step, no npm, no separate server.

The blueprint illustrates three characteristic concerns of a fullstack Akka application:

- **Durable conversation state** — `ConversationEntity` stores every `UserMessageAdded` and `AgentReplyRecorded` event. Replaying events reconstructs the full history. A browser refresh fetches the same conversation from the entity.
- **Streaming UI** — `ConversationView` projects entity events into a read-model row that the endpoint serves via Server-Sent Events. The UI never polls; the SSE stream delivers each state transition as it happens.
- **Bounded per-turn lifecycle** — `ConversationWorkflow` gives each turn its own workflow instance with explicit step timeouts, so a slow or failing LLM call cannot block the conversation entity indefinitely.

The blueprint is intentionally general: the agent has no domain-specific restrictions; a deployer who needs content-safety rules, PII filters, or topic guardrails would add those controls to `eval-matrix.yaml` before generating.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks an existing conversation from the left panel or clicks **New conversation** to create one. Creating a conversation POSTs to `/api/conversations`, receives a `conversationId`, and immediately shows an empty chat thread.
2. The user types a message in the input field and clicks **Send**. The UI POSTs to `/api/conversations/{id}/messages` and the message card appears immediately in the thread with status `AWAITING_REPLY`.
3. Within ~10–30 s, the workflow's `generateStep` completes. The card flips to `REPLY_RECORDED` and the agent's reply text appears below the user message. A small token-count chip shows how many tokens the model used.
4. The user continues typing. Each turn is independent; the workflow receives the last N messages as context.
5. The user can open a second browser tab and see the same conversation updating live — both tabs share the same SSE stream.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create, list, get, send message, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation message history: created → active. Append-only event log. | `ChatEndpoint`, `ConversationWorkflow` | `ConversationView` |
| `ConversationWorkflow` | `Workflow` | One workflow per message turn. Steps: `generateStep` → `recordStep`. | started by `ChatEndpoint` on each message | `ChatAgent`, `ConversationEntity` |
| `ChatAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the system persona as its definition instructions and the conversation context (recent history) as a task attachment; returns `AgentReply`. | invoked by `ConversationWorkflow` | returns reply |
| `ConversationView` | `View` | Read model: one row per conversation with its full message list. | `ConversationEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ChatMessage(
    String messageId,
    MessageRole role,           // USER or AGENT
    String content,
    Optional<Integer> tokenCount,  // present on AGENT messages
    Instant createdAt
) {}
enum MessageRole { USER, AGENT }

record SendMessageRequest(
    String conversationId,
    String userMessage,
    String submittedBy
) {}

record AgentReply(
    String content,
    int tokenCount,
    Instant repliedAt
) {}

record Conversation(
    String conversationId,
    String title,
    List<ChatMessage> messages,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> lastActivityAt
) {}

enum ConversationStatus {
    CREATED, ACTIVE, AWAITING_REPLY, REPLY_RECORDED, FAILED
}
```

Events on `ConversationEntity`: `ConversationCreated`, `UserMessageAdded`, `ReplyGenerationStarted`, `AgentReplyRecorded`, `ConversationFailed`.

Every nullable lifecycle field on `Conversation` is `Optional<T>` (Lesson 6). `tokenCount` on `ChatMessage` is `Optional<Integer>` because user messages carry no token count.

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ title, submittedBy }` → `{ conversationId }`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation with its full message list.
- `POST /api/conversations/{id}/messages` — body `{ userMessage, submittedBy }` → `{ messageId, workflowId }`.
- `GET /api/conversations/{id}/sse` — Server-Sent Events; one event per state transition on the conversation.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Gemini Fullstack Chat</title>`.

The App UI tab is a two-column layout: a left rail with the conversation list (title, last-message preview, age, status chip) and a right pane with the selected conversation's thread — a vertical list of message cards (user messages right-aligned, agent replies left-aligned) plus a text input and Send button at the bottom.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. This baseline has **no controls** — it is a general-purpose chat demonstrator with no domain-specific regulatory requirements. A deployer targeting a regulated sector (healthcare, finance, legal) would add controls covering PII handling, content safety, and output auditing before generating.

The `eval-matrix.yaml` contains an empty `controls` list and a matching empty `simplified_view` block to satisfy the schema.

## 9. Agent prompts

- `ChatAgent` → `prompts/chat-agent.md`. The single decision-making LLM. System prompt instructs it to act as a helpful, accurate assistant, respond only in the language of the user's message, and keep replies concise unless the user explicitly asks for more detail.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User creates a conversation and sends a message; within 30 s the agent reply appears in the thread with a non-empty `content` field and a token count.
2. **J2** — Two conversations run independently; messages from conversation A never appear in conversation B's thread.
3. **J3** — The SSE stream delivers state transitions in order (`AWAITING_REPLY` → `REPLY_RECORDED`) without the UI polling; the browser devtools show exactly one open SSE connection per conversation.
4. **J4** — After the reply lands, the user refreshes the page; fetching `GET /api/conversations/{id}` returns the full message list including both the user message and the agent reply, confirming durable persistence.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named gemini-fullstack demonstrating the single-agent × general cell. Runs out
of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-fullstack-chat. Java package io.akka.samples.geminifullstack. Akka 3.6.0.
HTTP port 9552.

Components to wire (exactly):

- 1 AutonomousAgent ChatAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/chat-agent.md>) and
  .capability(TaskAcceptance.of(GENERATE_REPLY).maxIterationsPerTask(2)). The task receives
  the conversation context (last N messages formatted as JSON) as a task ATTACHMENT named
  "conversation.json" (NOT as inline prompt text — Akka's TaskDef.attachment(name,
  contentBytes) is the canonical call). Output: AgentReply{content: String, tokenCount: int,
  repliedAt: Instant}. The agent has no guardrail in this baseline; a deployer can add one
  by registering a before-agent-response hook on the agent definition.

- 1 Workflow ConversationWorkflow per (conversationId + messageId) with two steps:
  * generateStep — emits ReplyGenerationStarted, then calls componentClient
    .forAutonomousAgent(ChatAgent.class, "chat-" + conversationId)
    .runSingleTask(TaskDef.instructions("Reply to the user's latest message.")
      .attachment("conversation.json", buildContextJson(conversation).getBytes()))
    — returns a taskId, then forTask(taskId).result(GENERATE_REPLY) to fetch the reply.
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(1)
    .failoverTo(ConversationWorkflow::error).
  * recordStep — calls ConversationEntity.recordReply(agentReply). WorkflowSettings.stepTimeout
    10s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId: String, title: String, messages: List<ChatMessage>,
  status: ConversationStatus, createdAt: Instant, lastActivityAt: Optional<Instant>}.
  ConversationStatus enum: CREATED, ACTIVE, AWAITING_REPLY, REPLY_RECORDED, FAILED.
  Events: ConversationCreated{title, submittedBy}, UserMessageAdded{message},
  ReplyGenerationStarted{messageId}, AgentReplyRecorded{messageId, reply},
  ConversationFailed{reason}. Commands: create, addUserMessage, markGenerating,
  recordReply, fail, getConversation. emptyState() returns Conversation.initial("") with an
  empty messages list and no commandContext() reference (Lesson 3). Every Optional<T> field
  uses Optional.empty() in initial state and Optional.of(...) inside the event-applier.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation). Table updater
  consumes ConversationEntity events. ONE query getAllConversations: SELECT * AS conversations
  FROM conversation_view. No WHERE status filter — Akka cannot auto-index enum columns
  (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ChatEndpoint at /api with POST /conversations (body {title, submittedBy}; mints
    conversationId; calls ConversationEntity.create; returns {conversationId}), GET
    /conversations (list from getAllConversations, sorted newest-first), GET /conversations/{id}
    (one row), POST /conversations/{id}/messages (body {userMessage, submittedBy}; mints
    messageId; calls ConversationEntity.addUserMessage; starts ConversationWorkflow; returns
    {messageId, workflowId}), GET /conversations/{id}/sse (Server-Sent Events forwarded from
    the view's stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files
    from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ChatTasks.java declaring one Task<R> constant: GENERATE_REPLY = Task.name("Generate reply")
  .description("Read the conversation context and produce an AgentReply.")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records ChatMessage, MessageRole, SendMessageRequest, AgentReply, Conversation,
  ConversationStatus.

- ConversationContextBuilder.java — a pure utility that takes a List<ChatMessage> and an
  int contextWindowSize and returns a UTF-8 JSON byte array of the last contextWindowSize
  messages. No LLM call. Documented in Javadoc on the class.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9552 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ChatAgent.definition() binds the
  configured provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded conversation
  starters: a general Q&A exchange (3 user + 3 agent turns), a coding help exchange (4 user +
  4 agent turns), and a brainstorming exchange (2 user + 2 agent turns). Each seed carries
  realistic user messages and plausible agent replies — used by the "Load seeded example" UI
  action.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with an empty controls list. No regulation_anchors —
  this is a general-purpose baseline.

- risk-survey.yaml at the project root with data.data_classes.pii = false (baseline
  assumption; deployer overrides), decisions.authority_level = assist-only (the agent's
  reply is informational), oversight.human_in_loop = false (the user reads replies directly);
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/chat-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Gemini Fullstack Chat", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  conversation list with status chips; right = selected conversation thread with user messages
  right-aligned, agent replies left-aligned, and a message input + Send button at the bottom).
  Browser title exactly: <title>Akka Sample: Gemini Fullstack Chat</title>. No subtitle on
  the Overview tab.

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
- NEVER write the key value to any file Akka creates. Akka records only the REFERENCE
  (env-var name, file path, secrets URI); the value lives in the user's existing
  infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime. The message must not echo any captured key
  material.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversationId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-reply.json — 10 AgentReply entries covering general, coding, and brainstorming
      content. Each entry has a non-empty content string (2–4 sentences), a realistic
      tokenCount between 40 and 200, and a repliedAt timestamp. Variety in tone and style
      so repeated calls don't look identical. The mock should select entries deterministically
      per (conversationId + messageId) so replaying seeds produces the same thread.
- A MockModelProvider.seedFor(conversationId, messageId) helper makes per-turn selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChatAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChatTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (generateStep 60s, recordStep 10s,
  error 5s).
- Lesson 6: every nullable lifecycle field on the Conversation record is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: ChatTasks.java with GENERATE_REPLY = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9552 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ChatAgent). No second
  agent is introduced.
- The conversation context is passed as a Task ATTACHMENT, never inlined into the agent's
  instructions. Verify the generated generateStep uses TaskDef.attachment(...) and not string
  interpolation into the instruction text.
- The Overview tab's Try-it card shows just "/akka:build" — no env-var export block. Per
  Lesson 25, /akka:specify handles the key during generation.
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
