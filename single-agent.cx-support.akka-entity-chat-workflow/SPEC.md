# SPEC — entity-workflow-chat

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Entity Workflow Chat.
**One-line pitch:** Each customer-support conversation is a long-lived Workflow paired with an EventSourcedEntity — the entity holds the complete turn log, the workflow drives each turn through sanitize → agent → record, and a continue-as-new compaction step keeps the live context window bounded while preserving the full history for audit.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the customer-support domain. One `ConversationAgent` (AutonomousAgent) produces every reply; the surrounding entity and workflow only manage the conversation lifecycle, sanitize inputs, and compact history. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw user message and the agent call — so the model never sees customer identifiers. The raw message is kept on the entity for audit; the sanitized form is what the agent sees in its context window.
- A **human-in-the-loop gate** where every user turn is the explicit trigger for the next workflow step. There is no background polling loop. The user sending a message IS the control signal that advances the conversation — each `POST /api/conversations/{id}/messages` resumes the paused workflow from `awaitTurnStep`.

The blueprint also demonstrates the continue-as-new compaction pattern: when the accumulated turn context exceeds 4 000 tokens, the workflow's `compactStep` produces a `ConversationSummary`, emits `HistoryCompacted` on the entity, and the next `agentStep` receives the summary plus only the most recent turns rather than the full log.

## 3. User-facing flows

The user opens the App UI tab.

1. The user fills a **Topic** field (brief free-text description of their support issue) and clicks **Start conversation**. The UI POSTs to `/api/conversations` and receives a `conversationId`.
2. The new conversation card appears in the live list with status `OPEN`. The right pane shows an empty message thread and a text input.
3. The user types a message and clicks **Send**. The UI POSTs to `/api/conversations/{id}/messages`. Within ~1 s the card's status shows `AGENT_THINKING`.
4. Within ~10–30 s the agent reply appears in the thread. Status returns to `OPEN` (waiting for the next user turn).
5. The user continues the conversation across multiple turns. When the token budget is exceeded, the system compacts history silently — the conversation continues without interruption; a small `compacted` chip appears on older turns indicating they have been summarised.
6. The user clicks **Close conversation**. Status transitions to `CLOSED`. The full turn log remains accessible via `GET /api/conversations/{id}`.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/conversations/*` — create, send message, close, list, get, SSE; serves `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-conversation state: full turn log, compaction summaries, status. Source of truth. | `ChatEndpoint`, `MessageSanitizer`, `ChatWorkflow` | `ConversationView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; strips PII; calls `ConversationEntity.attachSanitized`. | `ConversationEntity` events | `ConversationEntity` |
| `ChatWorkflow` | `Workflow` | One workflow per conversation. Steps: `awaitTurnStep` → `agentStep` → `recordStep` → `compactStep` (conditional). Pauses at `awaitTurnStep` between turns. | resumed by user messages via `ChatEndpoint` | `ConversationAgent`, `ConversationEntity` |
| `ConversationAgent` | `AutonomousAgent` | The one decision-making LLM. Receives a context package (sanitized history + new sanitized message) as a task attachment; returns `AgentReply`. | invoked by `ChatWorkflow.agentStep` | returns reply |
| `ConversationView` | `View` | Read model: one row per conversation for the UI. | `ConversationEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record UserMessage(
    String messageId,
    String rawText,
    Instant sentAt
) {}

record SanitizedMessage(
    String messageId,
    String redactedText,
    List<String> piiCategoriesFound,
    Instant sanitizedAt
) {}

record AgentReply(
    String messageId,
    String replyText,
    boolean suggestClose,
    Instant repliedAt
) {}

record Turn(
    String turnId,
    SanitizedMessage userMessage,
    Optional<AgentReply> agentReply,
    TurnStatus status
) {}
enum TurnStatus { WAITING_SANITIZE, READY, AGENT_THINKING, COMPLETED, FAILED }

record ConversationSummary(
    String summaryText,
    int turnsCompacted,
    Instant compactedAt
) {}

record Conversation(
    String conversationId,
    String topic,
    String userId,
    List<Turn> turns,
    List<ConversationSummary> summaries,
    ConversationStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum ConversationStatus {
    OPEN, AGENT_THINKING, COMPACTING, CLOSED, FAILED
}
```

Events on `ConversationEntity`: `ConversationStarted`, `MessageReceived`, `MessageSanitized`, `AgentReplied`, `HistoryCompacted`, `ConversationClosed`, `ConversationFailed`.

Every nullable lifecycle field on the `Conversation` state record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/conversations` — body `{ topic, userId }` → `{ conversationId }`.
- `POST /api/conversations/{id}/messages` — body `{ text }` → `{ messageId }`. Resumes the paused workflow.
- `POST /api/conversations/{id}/close` — body `{}` → `204`.
- `GET /api/conversations` — list all conversations, newest-first.
- `GET /api/conversations/{id}` — one conversation with full turn log.
- `GET /api/conversations/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Entity Workflow Chat</title>`.

The App UI tab is a two-column layout: a left rail with the live list of open and closed conversations (status pill + topic + age) and a right pane with the selected conversation's full thread — turn-by-turn messages, PII-redacted user text, agent replies, compaction markers, and a close button.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): strips emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like identifiers from each incoming user message before the agent context is assembled. The raw message text is preserved on the entity turn log for support audit but is never part of the context window passed to the model.
- **H1 — Human-in-the-loop gate** (`hitl`, `application`): every agent call is triggered by an explicit user message delivery. The workflow pauses in `awaitTurnStep` after each reply and does not advance until the user sends the next message via `POST /api/conversations/{id}/messages`. There is no autonomous loop — the user controls the conversation cadence.

## 9. Agent prompts

- `ConversationAgent` → `prompts/conversation-agent.md`. The single decision-making LLM. System prompt establishes the agent's customer-support role, instructs it to read the conversation context attachment and produce a `AgentReply`, and specifies when to set `suggestClose = true`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User starts a conversation on a billing issue, sends three turns; each reply appears with the correct `replyText` and `suggestClose` field; after three turns the full turn log on the entity has 3 completed turns.
2. **J2** — A long conversation exceeds 4 000 tokens; `compactStep` fires and produces a `ConversationSummary`; the turn after compaction shows that the agent's reply correctly references the summarised context; a `compacted` chip marks the boundary in the UI.
3. **J3** — A user message containing `john.smith@example.com` and `card 4111-1111-1111-1111` is sent; the agent context attachment contains only `[REDACTED-EMAIL]` and `[REDACTED-PCN]`; the entity's raw turn log retains the originals.
4. **J4** — The workflow is paused in `awaitTurnStep`; no background agent call fires until a new user message arrives; confirmed by absence of `AgentReplied` events between user messages.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named entity-workflow-chat demonstrating the single-agent × cx-support cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-akka-entity-chat-workflow. Java package
io.akka.samples.entityworkflowchat. Akka 3.6.0. HTTP port 9134.

Components to wire (exactly):

- 1 AutonomousAgent ConversationAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/conversation-agent.md>) and
  .capability(TaskAcceptance.of(REPLY_TO_TURN).maxIterationsPerTask(3)). The task receives
  the conversation context (prior compacted summary + recent sanitized turns) as a task
  ATTACHMENT named "context.json" (NOT as inline prompt text — Akka's
  TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  AgentReply{messageId: String, replyText: String, suggestClose: boolean, repliedAt: Instant}.

- 1 Workflow ChatWorkflow per conversationId with four steps:
  * awaitTurnStep — pauses workflow (uses Workflow.pause()) until a sanitized user message is
    available. The workflow is resumed externally by ChatEndpoint when the user POSTs a new
    message. On resume advances to agentStep. WorkflowSettings.stepTimeout 300s (the wait is
    bounded by a session idle limit, not a wall clock on agent latency).
  * agentStep — assembles the context package (JSON object: {summary: String | null,
    recentTurns: List<SanitizedTurn>}) from ConversationEntity.getConversation(), passes it
    as attachment "context.json"; calls componentClient.forAutonomousAgent(
    ConversationAgent.class, "agent-" + conversationId + "-" + turnId).runSingleTask(
      TaskDef.instructions("Reply to the next user message.")
        .attachment("context.json", contextBytes)
    ); fetches result via forTask(taskId).result(REPLY_TO_TURN). On success calls
    ConversationEntity.recordReply(reply). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(ChatWorkflow::error).
  * recordStep — calls ConversationEntity.completeTurn(turnId). Checks whether accumulated
    token count exceeds 4000 (rough estimate: total chars / 4). If yes, transitions to
    compactStep; otherwise loops back to awaitTurnStep. WorkflowSettings.stepTimeout 5s.
  * compactStep — runs ContextCompactor (a deterministic in-process summariser — NOT an LLM
    call). ContextCompactor accepts the list of completed Turn records and returns a
    ConversationSummary (summaryText: a bullet-list of resolved topics and open items,
    turnsCompacted: int, compactedAt: Instant). Calls
    ConversationEntity.recordCompaction(summary). Then loops back to awaitTurnStep.
    WorkflowSettings.stepTimeout 10s.
  * error step calls ConversationEntity.fail(reason) and transitions to a terminal state.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per conversationId). State
  Conversation{conversationId: String, topic: String, userId: String, turns: List<Turn>,
  summaries: List<ConversationSummary>, status: ConversationStatus, createdAt: Instant,
  closedAt: Optional<Instant>}. ConversationStatus enum: OPEN, AGENT_THINKING, COMPACTING,
  CLOSED, FAILED. Events: ConversationStarted{topic, userId}, MessageReceived{userMessage},
  MessageSanitized{messageId, sanitized}, AgentReplied{reply}, HistoryCompacted{summary},
  ConversationClosed{closedAt}, ConversationFailed{reason}. Commands: start, receiveMessage,
  attachSanitized, recordReply, completeTurn, recordCompaction, close, fail,
  getConversation. emptyState() returns Conversation.initial("") with all list fields as
  empty lists and closedAt = Optional.empty() (Lesson 3). Every Optional<T> field uses
  Optional.empty() in initial state and Optional.of(...) inside event appliers.

- 1 Consumer MessageSanitizer subscribed to ConversationEntity events; on MessageReceived
  runs a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like,
  payment-card-like, postal addresses, person-name heuristic, account-id-like tokens) over
  rawText, computes the list of categories found, builds SanitizedMessage, then calls
  ConversationEntity.attachSanitized(messageId, sanitized). After attachSanitized lands, the
  same Consumer resumes ChatWorkflow with id = "chat-" + conversationId by calling the
  workflow's resume endpoint with the sanitized message id.

- 1 View ConversationView with row type ConversationRow (mirrors Conversation minus raw
  message text in turns — the audit log keeps the raw; the view holds sanitized forms).
  Table updater consumes ConversationEntity events. ONE query getAllConversations:
  SELECT * AS conversations FROM conversation_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ChatEndpoint at /api with:
    POST /conversations (body {topic, userId}; mints conversationId; calls
      ConversationEntity.start; starts ChatWorkflow with id "chat-" + conversationId;
      returns {conversationId}),
    POST /conversations/{id}/messages (body {text}; mints messageId; calls
      ConversationEntity.receiveMessage; returns {messageId}),
    POST /conversations/{id}/close (calls ConversationEntity.close; returns 204),
    GET /conversations (list from getAllConversations, newest-first),
    GET /conversations/{id} (one row),
    GET /conversations/sse (Server-Sent Events forwarded from view stream-updates),
    three /api/metadata/* endpoints serving the YAML/MD files from
      src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ChatTasks.java declaring one Task<R> constant: REPLY_TO_TURN = Task.name("Reply to turn")
  .description("Read the conversation context attachment and produce an AgentReply for the
  latest user message").resultConformsTo(AgentReply.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records UserMessage, SanitizedMessage, AgentReply, Turn, TurnStatus,
  ConversationSummary, Conversation, ConversationStatus.

- ContextCompactor.java — pure deterministic logic (no LLM). Inputs: List<Turn>. Outputs:
  ConversationSummary. Algorithm: extracts each completed turn's topic signal (first 80
  chars of sanitized message + first 80 chars of reply), groups by keyword overlap, emits a
  bullet list. Javadoc on class describes the algorithm.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9134 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ConversationAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded support
  scenarios: billing dispute (5 turns), password reset (3 turns), order status inquiry
  (4 turns). Each seed contains 2–3 plausible PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, H1) matching the mechanisms
  in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = advisory-response
  (the agent's replies are informational, never binding), oversight.human_in_loop = true
  (the user drives every turn); deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/conversation-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Entity Workflow Chat", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of conversation cards; right = selected-conversation thread with message
  bubbles, PII chips, compaction markers, agent reply bubbles, and a Close button).
  Browser title exactly: <title>Akka Sample: Entity Workflow Chat</title>. No subtitle on
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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(conversationId +
  turnId)), and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    reply-to-turn.json — 10 AgentReply entries covering the three seeded scenarios
      (billing dispute / password reset / order status) across multiple turns. Each entry
      has a non-empty replyText (1–3 sentences appropriate to the support topic) and a
      realistic suggestClose boolean (true only on final resolution turns). Include 2
      entries where replyText is an empty string — these are only used by the compaction
      test (the agent returns an empty reply for a compacted turn boundary marker). The mock
      should use seedFor(conversationId + turnId) so the same turn always returns the same
      mock response.
- A MockModelProvider.seedFor(key) helper makes per-turn selection deterministic across
  restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ConversationAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChatTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (agentStep
  60s, awaitTurnStep 300s, recordStep 5s, compactStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on the Conversation state record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: ChatTasks.java with REPLY_TO_TURN = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9134 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words — shape, minimal, smaller, complex, Akka SDK in
  narrative, marketing tone, competitor brand names.
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ConversationAgent). The
  context compaction is rule-based (ContextCompactor.java) and does NOT make an LLM call —
  keeping the pattern's "one agent" promise honest.
- The user message is passed as a task ATTACHMENT alongside conversation context, never
  inlined into the agent's instructions. Verify the generated agentStep uses
  TaskDef.attachment(...) and not string interpolation into the instruction text.
- The human-in-the-loop gate is structural: the workflow genuinely pauses at awaitTurnStep
  and does not advance until a user message arrives. There is no timer or autonomous
  re-entry that bypasses the user.
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
