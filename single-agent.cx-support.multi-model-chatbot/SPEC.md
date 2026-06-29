# SPEC — multi-model-chatbot

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MultiModelChatbot.
**One-line pitch:** A user sends a support message; one AI agent selects a reply using the configured LLM provider (switchable at runtime) and streams the reply back — every turn stored in an event-sourced conversation log, PII scrubbed before the model sees it, and harmful replies blocked by a `before-agent-response` guardrail before they ever leave the agent loop.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the cx-support domain. One `ChatAgent` (AutonomousAgent) produces every reply; the surrounding components sanitize its input and moderate its output. Two governance mechanisms are wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw message submission and the agent call — so the model never sees real names, email addresses, phone numbers, or account identifiers.
- A **before-agent-response guardrail** (`ReplyGuardrail`) validates each candidate reply on every turn: no harmful content, no off-topic brand references, no re-exposure of the PII that was redacted, and the reply must parse as a well-formed `ChatReply` JSON object. A rejected reply triggers a retry inside the same task.

The blueprint shows that the single-agent pattern does not mean "ungoverned" — two independent checks sit on either side of the one decision-making LLM call, and the event-sourced entity preserves the full audit trail for every session.

## 3. User-facing flows

The user opens the App UI tab.

1. The user either starts a new conversation (which mints a `sessionId`) or picks an existing session from the left-rail session list.
2. The user types a message into the **Message** input and presses **Send**. The UI POSTs to `/api/chat/{sessionId}/messages` and the message card appears immediately with status `RECEIVED`.
3. Within ~1 s, the card transitions to `SANITIZED`. A small badge shows which PII categories (if any) were redacted. The sanitized text is visible in the card detail.
4. Within ~5–30 s (depending on provider latency), the card transitions to `REPLIED`. The reply text appears in the chat bubble, attributed with the provider name (e.g., `claude-sonnet-4-6`) and a latency indicator.
5. The user can send a follow-up message in the same session. The agent receives the full conversation history as context.
6. From the **Settings** panel, the user can switch the active provider (Anthropic / OpenAI / Google Gemini). The next message in the same session uses the new provider; the provider name is visible on the reply bubble.
7. The user can start multiple sessions; the left-rail list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ChatEndpoint` | `HttpEndpoint` | `/api/chat/*` — start session, send message, list sessions, get session, SSE; `/api/metadata/*`. | — | `ConversationEntity`, `ConversationView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `ConversationEntity` | `EventSourcedEntity` | Per-session lifecycle: one entity per `sessionId`. Stores all message turns and provider selection. Source of truth. | `ChatEndpoint`, `MessageSanitizer`, `ChatWorkflow` | `ConversationView` |
| `MessageSanitizer` | `Consumer` | Subscribes to `MessageReceived` events; redacts PII from `userText`; calls `ConversationEntity.attachSanitized`. | `ConversationEntity` events | `ConversationEntity` |
| `ChatWorkflow` | `Workflow` | One workflow instance per message turn. Steps: `awaitSanitizedStep` → `replyStep`. | started by `MessageSanitizer` once sanitized event lands | `ChatAgent`, `ConversationEntity` |
| `ChatAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sanitized message and conversation history as task instructions; returns `ChatReply`. | invoked by `ChatWorkflow` | returns reply |
| `ReplyGuardrail` | guardrail class | Registered on `ChatAgent` via `before-agent-response` hook. Checks reply for harmful content, off-topic brand names, PII re-exposure, and parseable `ChatReply` shape. | `ChatAgent` loop | pass-through or rejection |
| `ConversationView` | `View` | Read model: one row per session for the UI list. | `ConversationEntity` events | `ChatEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record MessageTurn(
    String turnId,
    String userText,
    @Nullable String sanitizedText,
    List<String> piiCategoriesFound,
    @Nullable ChatReply reply,
    TurnStatus status,
    Instant receivedAt,
    @Nullable Instant repliedAt
) {}

record ChatReply(
    String replyText,
    String providerName,
    String modelId,
    int inputTokens,
    int outputTokens,
    Instant generatedAt
) {}

enum TurnStatus { RECEIVED, SANITIZED, REPLIED, FAILED }

record ConversationSession(
    String sessionId,
    String userId,
    String displayName,
    String activeProvider,  // "anthropic" | "openai" | "googleai-gemini"
    List<MessageTurn> turns,
    SessionStatus status,
    Instant createdAt,
    @Nullable Instant lastActivityAt
) {}

enum SessionStatus { ACTIVE, CLOSED }
```

Events on `ConversationEntity`: `SessionStarted`, `MessageReceived`, `MessageSanitized`, `ReplyGenerated`, `TurnFailed`, `ProviderChanged`, `SessionClosed`.

Every nullable lifecycle field on `MessageTurn` and `ConversationSession` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/chat/sessions` — body `{ userId, displayName, activeProvider? }` → `{ sessionId }`.
- `POST /api/chat/{sessionId}/messages` — body `{ userText, requestedProvider? }` → `{ turnId }`.
- `GET /api/chat/sessions` — list all sessions, newest-first.
- `GET /api/chat/{sessionId}` — one session with all turns.
- `GET /api/chat/{sessionId}/sse` — Server-Sent Events; one event per turn state transition.
- `PUT /api/chat/{sessionId}/provider` — body `{ provider }` → `204`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MultiModelChatbot</title>`.

The App UI tab is a two-column layout: a left rail with the session list (newest-first, with session display name and last-activity age) and a right pane with the selected session's chat thread — message bubbles, turn status badges, PII category chips, provider attribution on each reply bubble, and a message input at the bottom.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail** (`ReplyGuardrail`): runs on every turn of `ChatAgent`. Asserts the candidate response is well-formed `ChatReply` JSON; checks `replyText` against a content-policy pattern set (harmful language, explicit threats, PII patterns that should have been redacted); blocks any reply that re-exposes a PII token from the redacted input. On failure, returns a structured `invalid-response` error to the agent loop so the task retries within its iteration budget.
- **S1 — PII sanitizer** (`pii`, applied inside `MessageSanitizer` Consumer): redacts email addresses, phone numbers, person names, postal addresses, account-id-like tokens, and government-identifier patterns from the user's raw message before any LLM call. Records which categories were found. The raw text is preserved on the entity for audit.

## 9. Agent prompts

- `ChatAgent` → `prompts/chat-agent.md`. The single decision-making LLM. System prompt instructs it to act as a helpful support agent, answer within the product's scope, and return a well-formed `ChatReply` JSON object.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User starts a session, sends a support message → the turn transitions through SANITIZED → REPLIED within 30 s; the reply bubble shows the provider name and a non-empty `replyText`.
2. **J2** — The agent's first response on a turn is policy-violating (mock LLM path) — the `before-agent-response` guardrail rejects it; the second iteration produces a clean reply; the UI never displays the rejected content.
3. **J3** — A message containing `alice@corp.com` and `555-867-5309` is sent; the LLM call log shows only `[REDACTED-EMAIL]` and `[REDACTED-PHONE]`; the entity's raw `userText` retains the originals for audit.
4. **J4** — User switches the active provider mid-session; the next reply bubble is attributed to the new provider; prior turns retain their original provider attribution.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named multi-model-chatbot demonstrating the single-agent × cx-support cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-cx-support-multi-model-chatbot. Java package io.akka.samples.nextjsaichatbot.
Akka 3.6.0. HTTP port 9994.

Components to wire (exactly):

- 1 AutonomousAgent ChatAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/chat-agent.md>) and
  .capability(TaskAcceptance.of(GENERATE_REPLY).maxIterationsPerTask(3)). The task receives
  the sanitized message and conversation history as its instruction text (NOT as a task
  attachment — the history is small enough to be inline). Output:
  ChatReply{replyText: String, providerName: String, modelId: String, inputTokens: int,
  outputTokens: int, generatedAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow ChatWorkflow per (sessionId, turnId) with two steps:
  * awaitSanitizedStep — polls ConversationEntity.getSession every 1s; on the matching
    turn's status == SANITIZED advances to replyStep. WorkflowSettings.stepTimeout 15s.
  * replyStep — calls componentClient.forAutonomousAgent(ChatAgent.class,
    "chat-" + sessionId).runSingleTask(
      TaskDef.instructions(formatTurnContext(session, turnId))
    ) — returns a taskId, then forTask(taskId).result(GENERATE_REPLY) to fetch the reply.
    On success calls ConversationEntity.recordReply(turnId, reply).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(ChatWorkflow::error).
  error step transitions the turn to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity ConversationEntity (one per sessionId). State
  ConversationSession{sessionId: String, userId: String, displayName: String,
  activeProvider: String, turns: List<MessageTurn>, status: SessionStatus,
  createdAt: Instant, lastActivityAt: Optional<Instant>}.
  SessionStatus enum: ACTIVE, CLOSED. TurnStatus enum: RECEIVED, SANITIZED, REPLIED, FAILED.
  Events: SessionStarted{session}, MessageReceived{turnId, userText, receivedAt},
  MessageSanitized{turnId, sanitizedText, piiCategoriesFound}, ReplyGenerated{turnId, reply},
  TurnFailed{turnId, reason}, ProviderChanged{newProvider}, SessionClosed{}.
  Commands: startSession, receiveMessage, attachSanitized, recordReply, failTurn,
  changeProvider, closeSession, getSession.
  emptyState() returns ConversationSession.initial("") with empty turns list and no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside the event-applier.

- 1 Consumer MessageSanitizer subscribed to ConversationEntity events; on MessageReceived
  runs a regex+heuristic redaction pipeline (emails, phone numbers, person names, postal
  addresses, account-id-like tokens, government-identifier patterns) over userText, computes
  the list of categories found, then calls ConversationEntity.attachSanitized(turnId,
  sanitizedText, piiCategoriesFound). After attachSanitized lands, the same Consumer starts
  a ChatWorkflow with id = "chat-" + sessionId + "-" + turnId.

- 1 View ConversationView with row type SessionRow (mirrors ConversationSession with full
  turns list). Table updater consumes ConversationEntity events. ONE query getAllSessions:
  SELECT * AS sessions FROM conversation_view. No WHERE status filter — Akka cannot
  auto-index enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * ChatEndpoint at /api with:
    POST /chat/sessions (body {userId, displayName, activeProvider?}; mints sessionId;
      calls ConversationEntity.startSession; returns {sessionId}),
    POST /chat/{sessionId}/messages (body {userText, requestedProvider?}; mints turnId;
      calls ConversationEntity.receiveMessage; returns {turnId}),
    GET /chat/sessions (list from getAllSessions, sorted newest-first),
    GET /chat/{sessionId} (one session from view),
    GET /chat/{sessionId}/sse (Server-Sent Events from view stream-updates),
    PUT /chat/{sessionId}/provider (body {provider}; calls ConversationEntity.changeProvider;
      returns 204),
    and three /api/metadata/* endpoints serving YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- ChatTasks.java declaring one Task<R> constant: GENERATE_REPLY = Task.name("Generate reply")
  .description("Read the conversation history and produce a ChatReply for the latest user message")
  .resultConformsTo(ChatReply.class). DO NOT skip this — the AutonomousAgent requires its
  companion Tasks class (Lesson 7).

- Domain records MessageTurn, ChatReply, TurnStatus, ConversationSession, SessionStatus.

- ReplyGuardrail.java implementing the before-agent-response hook. Reads the candidate
  ChatReply from the LLM response, runs the checks listed in eval-matrix.yaml G1 (parseable
  JSON, no harmful-content patterns, no re-exposed PII tokens), and either passes the
  response through or returns Guardrail.reject(<structured-error>) to force the agent loop
  to retry.

- ContentPolicyChecker.java — pure deterministic logic (no LLM). Accepts replyText plus
  the list of redacted PII tokens from the session turn. Checks harmful-language patterns
  and PII re-exposure. Returns either pass() or a structured CheckResult naming the failed
  policy rule. Used by ReplyGuardrail.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9994 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ChatAgent.definition() binds the
  configured provider via .modelProvider("${akka.javasdk.agent.default}") or the per-agent
  override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded support
  conversation starters: a billing inquiry, a feature request, and a technical support
  question. Each starter message contains 1–2 plausible PII strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, S1) matching the mechanisms in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true, decisions.authority_level = autonomous-action
  (the agent's reply is sent directly to the user), oversight.human_in_loop = false,
  oversight.human_on_loop = true (support supervisor can review flagged sessions),
  failure.failure_modes including "harmful-reply", "pii-leakage-via-llm",
  "off-topic-response", "provider-unavailable"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/chat-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: MultiModelChatbot", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  session list + new-session button; right = chat thread with message bubbles, PII category
  chips on user turns, provider attribution on reply bubbles, message input at bottom, and
  provider selector in the session header).
  Browser title exactly: <title>Akka Sample: MultiModelChatbot</title>. No subtitle on the
  Overview tab.

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
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId, turnId)),
  and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    generate-reply.json — 8 ChatReply entries covering a range of support scenarios
    (billing, feature, technical). Each entry has a non-empty replyText, a providerName
    ("anthropic-mock"), a modelId ("mock-claude"), and plausible token counts. Plus 2
    deliberately MALFORMED entries (one with replyText containing a harmful-content trigger
    phrase; one that is not parseable JSON) — the guardrail blocks both, exercising the retry
    path. The mock should select a malformed entry on the FIRST iteration of every 3rd turn
    (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(sessionId, turnId) helper makes per-turn selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ChatAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion ChatTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (replyStep
  60s, awaitSanitizedStep 15s, error 5s).
- Lesson 6: every nullable lifecycle field on MessageTurn and ConversationSession is
  Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: ChatTasks.java with GENERATE_REPLY = Task.name(...).description(...)
  .resultConformsTo(ChatReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9994 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (ChatAgent). The content
  policy checker (ContentPolicyChecker.java) is deterministic and rule-based — NOT an LLM
  call — keeping the pattern's "one agent" promise honest.
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
