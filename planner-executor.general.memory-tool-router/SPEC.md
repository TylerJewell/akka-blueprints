# SPEC — memory-tool-router

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** AI Chatbot with Long-Term Memory and Dynamic Tool Routing.
**One-line pitch:** Each user turn is classified by a RouterAgent, executed by the matching tool-executor agent (calculator, knowledge base, web lookup, or code runner), and committed to a per-session memory store — with every tool call gated by a dispatch guardrail and every memory write scrubbed of PII.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern where the planner is a **RouterAgent** that classifies each turn and produces a `TurnPlan`, and the executors are four single-purpose tool-executor agents. A `MemoryAgent` sits alongside the loop to maintain long-term memory: it reads the session's memory before the router decides, and writes new memories after the tool result is available.

The blueprint demonstrates two governance mechanisms wired into that loop:

- a **before-tool-call guardrail** that vets each `RoutingDecision` against the registered tool allowlist and a per-tool content policy before the tool-executor agent is invoked,
- a **PII sanitizer** that scrubs personal identifiers, email addresses, phone numbers, and national-ID-shaped strings from every memory item before it is persisted to `MemoryEntity`.

## 3. User-facing flows

The user opens the App UI tab and types a message into the conversation input.

1. The system appends a `MessageReceived` event to `MessageQueue` and creates or resumes a `ConversationEntity` in `PROCESSING`.
2. A `ConversationWorkflow` turn starts. The `MemoryAgent` reads the session's current `MemorySnapshot` from `MemoryEntity` and returns a `MemoryContext`.
3. The `RouterAgent` receives the message text plus the `MemoryContext`. It produces a `TurnPlan { routingDecision, memoryUpdates }`.
4. The **before-tool-call guardrail** vets `routingDecision`. On rejection the workflow records a `TurnBlocked` entry and the `RouterAgent` is asked to revise (direct reply with no tool).
5. The matched tool-executor agent runs and returns a `ToolResult`.
6. The **PII sanitizer** scrubs the new `memoryUpdates` entries in the `TurnPlan`. Only the sanitized entries are written to `MemoryEntity`.
7. The workflow emits `TurnCompleted` on `ConversationEntity` with the reply text and sanitized memories.
8. `ConversationView` projects the update; the UI receives it via SSE.

A `SessionSimulator` (TimedAction) drips a sample conversation turn every 90 seconds so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RouterAgent` | `AutonomousAgent` | Classifies each user turn into a `TurnPlan`; also produces a revised `TurnPlan` when a dispatch is blocked. | `ConversationWorkflow` | returns `TurnPlan` to workflow |
| `MemoryAgent` | `AutonomousAgent` | Reads and summarises the session's `MemorySnapshot`; also identifies new facts from a turn result for the memory store. | `ConversationWorkflow` | returns `MemoryContext` or `MemoryDelta` |
| `CalculatorAgent` | `AutonomousAgent` | Evaluates mathematical expressions from seeded fixtures. | `ConversationWorkflow` | — |
| `KnowledgeBaseAgent` | `AutonomousAgent` | Answers factual questions from seeded knowledge fixtures. | `ConversationWorkflow` | — |
| `WebLookupAgent` | `AutonomousAgent` | Returns current-event answers from seeded web snippets. | `ConversationWorkflow` | — |
| `CodeRunnerAgent` | `AutonomousAgent` | Returns simulated execution output for an allow-listed expression set. | `ConversationWorkflow` | — |
| `ConversationWorkflow` | `Workflow` | Drives the receive → recall → route-and-guard → execute → sanitize → store-memory → reply loop plus halt and stuck branches. | `ConversationEndpoint`, `MessageQueueConsumer` | `ConversationEntity`, `MemoryEntity` |
| `ConversationEntity` | `EventSourcedEntity` | Holds the conversation's lifecycle, message history, and turn log. | `ConversationWorkflow` | `ConversationView` |
| `MemoryEntity` | `EventSourcedEntity` | Holds the per-session structured memory store (list of `MemoryItem`). Keyed by `sessionId`. | `ConversationWorkflow` | `ConversationView` |
| `SystemControlEntity` | `EventSourcedEntity` | Holds the operator halt flag. Single instance keyed by literal `"global"`. | `ConversationEndpoint` (operator action) | `ConversationWorkflow` (polls) |
| `MessageQueue` | `EventSourcedEntity` | Audit log of submitted messages. | `ConversationEndpoint`, `SessionSimulator` | `MessageQueueConsumer` |
| `ConversationView` | `View` | List-of-conversations read model for the UI. | `ConversationEntity` events | `ConversationEndpoint` |
| `MessageQueueConsumer` | `Consumer` | Subscribes to `MessageQueue` events; starts a `ConversationWorkflow` turn per submission. | `MessageQueue` events | `ConversationWorkflow` |
| `SessionSimulator` | `TimedAction` | Every 90 s, reads a line from `sample-events/session-prompts.jsonl` and enqueues it. | scheduler | `MessageQueue` |
| `StuckConversationMonitor` | `TimedAction` | Every 30 s, marks any conversation stuck in `PROCESSING` past 4 minutes as `STUCK`. | scheduler | `ConversationEntity` |
| `ConversationEndpoint` | `HttpEndpoint` | `/api/conversations/*` — submit, get, list, SSE, operator halt. | — | `ConversationView`, `MessageQueue`, `ConversationEntity`, `SystemControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record MessageRequest(String sessionId, String text, String sentBy) {}

record MemoryItem(
    String key,
    String value,
    MemoryKind kind,
    Instant recordedAt
) {}

record MemorySnapshot(
    String sessionId,
    List<MemoryItem> items
) {}

record MemoryContext(
    String sessionId,
    List<String> relevantFacts,
    int totalMemoryItems
) {}

record MemoryDelta(
    List<MemoryItem> toAdd,
    List<String> toForget
) {}

record RoutingDecision(
    ToolKind tool,
    String toolQuery,
    String rationale
) {}

record TurnPlan(
    RoutingDecision routingDecision,
    List<MemoryItem> memoryUpdates
) {}

record ToolResult(
    ToolKind tool,
    String query,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record TurnEntry(
    int turnNumber,
    String messageText,
    ToolKind toolUsed,
    TurnVerdict verdict,
    String reply,
    Optional<String> blocker,
    Instant completedAt
) {}

record TurnLog(List<TurnEntry> entries) {}

record Conversation(
    String conversationId,
    String sessionId,
    ConversationStatus status,
    List<String> messageHistory,
    Optional<TurnLog> turnLog,
    Optional<String> failureReason,
    Optional<String> haltReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ToolKind { CALCULATOR, KNOWLEDGE_BASE, WEB_LOOKUP, CODE_RUNNER, NONE }
enum MemoryKind { FACT, PREFERENCE, ENTITY_MENTION, CORRECTION }
enum TurnVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum ConversationStatus { IDLE, PROCESSING, HALTED, STUCK, ENDED }
```

### Events (`ConversationEntity`)

`ConversationStarted`, `MessageReceived`, `TurnStarted`, `TurnBlocked`, `TurnCompleted`, `TurnFailed`, `ConversationHaltedOperator`, `ConversationHaltedAutomatic`, `ConversationFailedTimeout`.

### Events (`MemoryEntity`)

`MemoryInitialised`, `MemoryItemsAdded`, `MemoryItemsForgotten`.

### Events (`SystemControlEntity`)

`HaltRequested`, `HaltCleared`.

### Events (`MessageQueue`)

`MessageSubmitted { conversationId, sessionId, text, sentBy, submittedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/conversations` — body `{ sessionId, text, sentBy? }` → `202 { conversationId }`. Enqueues message.
- `GET /api/conversations` — list all conversations. Optional `?status=...`.
- `GET /api/conversations/{id}` — one conversation (full turn log + message history).
- `GET /api/conversations/sse` — server-sent events stream of every conversation change.
- `GET /api/conversations/{id}/memory` — current `MemorySnapshot` for the conversation's `sessionId`.
- `POST /api/control/halt` — body `{ reason }` → `200`. Sets the operator halt flag.
- `POST /api/control/resume` — `200`. Clears the operator halt flag.
- `GET /api/control` — `{ halted, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "AI Chatbot with Long-Term Memory and Dynamic Tool Routing"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — conversation input form, operator halt/resume control, live list of conversations with status pills, expand-row to see the message history, the turn log, and the memory store.

Browser title: `<title>Akka Sample: AI Chatbot with Long-Term Memory and Dynamic Tool Routing</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present — state-diagram labels are otherwise invisible and arrow labels clip. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `RouterAgent`): every `RoutingDecision` is checked against (a) the registered tool allowlist (`CALCULATOR`, `KNOWLEDGE_BASE`, `WEB_LOOKUP`, `CODE_RUNNER`, `NONE`), (b) a per-tool content policy (CODE_RUNNER expressions may not call os.*, subprocess.*, or __import__; WEB_LOOKUP queries may not name hosts outside the allow-list; CALCULATOR expressions may not exceed 512 characters). Blocking. On rejection the workflow records a `TurnBlocked` entry and asks the `RouterAgent` to produce a direct reply with no tool.
- **S1 — PII sanitizer** (`sanitizer`, flavor `pii`): every `MemoryItem` produced by the `MemoryAgent` is scrubbed before it is written to `MemoryEntity`. The scrubber matches email addresses, phone numbers (E.164 and common national formats), national-ID shapes (SSN, UK NINO), and IP-address literals. Replacements are tagged: `[REDACTED:email]`, `[REDACTED:phone]`, `[REDACTED:national-id]`, `[REDACTED:ip-address]`. Raw PII never lands in the persistent memory store.

## 9. Agent prompts

- `RouterAgent` → `prompts/router.md`. Classifies turns; proposes tool; also produces no-tool fallback on block.
- `MemoryAgent` → `prompts/memory.md`. Reads/summarises memory; extracts facts from turn results.
- `CalculatorAgent` → `prompts/calculator.md`. Evaluates expressions from fixtures.
- `KnowledgeBaseAgent` → `prompts/knowledge-base.md`. Answers factual queries from fixtures.
- `WebLookupAgent` → `prompts/web-lookup.md`. Returns lookup answers from seeded snippets.
- `CodeRunnerAgent` → `prompts/code-runner.md`. Returns simulated execution output.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "What is 2+2 and what do you know about me?" for a fresh session. Conversation progresses `IDLE → PROCESSING`. Router dispatches `CALCULATOR`. Reply is "4" plus memory context. Memory store gains a new `FACT` item.
2. **J2** — Submit a message whose routing would invoke `CODE_RUNNER` with a forbidden expression (e.g., `__import__('os').listdir('/')`). The guardrail blocks the dispatch; the RouterAgent produces a direct reply with no tool; a `TurnBlocked` entry appears with `verdict = BLOCKED_BY_GUARDRAIL`.
3. **J3** — Submit a message containing a PII string (e.g., "My email is alice@example.com — remember that"). The `MemoryDelta` produced by `MemoryAgent` contains the raw email. The PII sanitizer replaces it with `[REDACTED:email]` before the `MemoryItemsAdded` event is emitted. The `MemoryEntity` state never contains the raw email.
4. **J4** — Click **Halt new dispatches** while a turn is `PROCESSING`. The in-flight tool call finishes; no further turns dispatch until Resume. The conversation ends in `HALTED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named memory-tool-router demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-memory-tool-router.
Java package io.akka.samples.aichatbotwithlongtermmemoryanddynamictoolrouting.
Akka 3.6.0. HTTP port 9428.

Components to wire (exactly):
- 6 AutonomousAgents:
  * RouterAgent — definition() with two capabilities:
      capability(TaskAcceptance.of(CLASSIFY_TURN).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(REVISE_TURN).maxIterationsPerTask(2)).
    System prompt from prompts/router.md. CLASSIFY_TURN returns TurnPlan.
    REVISE_TURN (called when guardrail blocks) returns TurnPlan with
    routingDecision.tool == NONE and a direct reply in memoryUpdates keyed
    "direct-reply".
  * MemoryAgent — two capabilities:
      capability(TaskAcceptance.of(RECALL).maxIterationsPerTask(2)) and
      capability(TaskAcceptance.of(EXTRACT_MEMORIES).maxIterationsPerTask(2)).
    Prompt from prompts/memory.md. RECALL returns MemoryContext.
    EXTRACT_MEMORIES returns MemoryDelta.
  * CalculatorAgent — capability(TaskAcceptance.of(EVALUATE).maxIterationsPerTask(2)).
    Prompt from prompts/calculator.md. Returns ToolResult.
  * KnowledgeBaseAgent — capability(TaskAcceptance.of(LOOK_UP).maxIterationsPerTask(2)).
    Prompt from prompts/knowledge-base.md. Returns ToolResult.
  * WebLookupAgent — capability(TaskAcceptance.of(WEB_FETCH).maxIterationsPerTask(2)).
    Prompt from prompts/web-lookup.md. Returns ToolResult.
  * CodeRunnerAgent — capability(TaskAcceptance.of(RUN_EXPRESSION).maxIterationsPerTask(2)).
    Prompt from prompts/code-runner.md. Returns ToolResult.

- 1 Workflow ConversationWorkflow with steps:
  recallStep -> routeStep -> guardrailStep -> executeStep ->
  extractMemoriesStep -> sanitizeMemoriesStep -> storeMemoriesStep ->
  replyStep -> checkHaltStep -> [back to recallStep for next turn OR
  to haltedStep / failStep].
  Step timeouts (override settings() per Lesson 4):
    recallStep ofSeconds(30), routeStep ofSeconds(45),
    executeStep ofSeconds(90) (covers any tool call),
    extractMemoriesStep ofSeconds(30), replyStep ofSeconds(30).
    defaultStepRecovery(maxRetries(2).failoverTo(ConversationWorkflow::error)).
  guardrailStep runs ToolDispatchGuardrail.vet(RoutingDecision); on reject
    records a TurnBlocked entry via ConversationEntity.recordBlock(turnNumber, reason)
    and calls RouterAgent.REVISE_TURN.
  executeStep switches on RoutingDecision.tool to call the matching agent via
    forAutonomousAgent(...).runSingleTask(...) then forTask(conversationId).result(...).
    On tool == NONE skips to extractMemoriesStep with empty ToolResult.
  sanitizeMemoriesStep applies PiiScrubber.scrub to every MemoryItem.value in
    the MemoryDelta. Replaces raw PII with tagged placeholders before storeMemoriesStep.
  storeMemoriesStep calls MemoryEntity(sessionId).addItems(scrubbed delta).
  checkHaltStep reads SystemControlEntity.get; on halted=true transitions to
    haltedStep (emits ConversationHaltedOperator on ConversationEntity).

- 1 EventSourcedEntity ConversationEntity holding Conversation state. Commands:
  startConversation, receiveMessage, recordTurnStart, recordBlock,
  recordTurnCompleted, recordTurnFailed, haltOperator, haltAutomatic,
  timeoutFail, getConversation. Events as listed in SPEC §5.

- 1 EventSourcedEntity MemoryEntity keyed by sessionId. State
  MemoryStore{String sessionId, List<MemoryItem> items}. Commands:
  initialise(sessionId), addItems(List<MemoryItem>), forgetItems(List<String> keys),
  getSnapshot, clear. Events: MemoryInitialised, MemoryItemsAdded, MemoryItemsForgotten.

- 1 EventSourcedEntity SystemControlEntity keyed by literal "global". State
  SystemControl{boolean halted, Optional<String> reason, Optional<Instant>
  haltedAt}. Commands: requestHalt(reason), clearHalt, get. Events:
  HaltRequested, HaltCleared.

- 1 EventSourcedEntity MessageQueue with command enqueueMessage(conversationId,
  sessionId, text, sentBy) emitting MessageSubmitted.

- 1 View ConversationView with row type ConversationRow (mirror of Conversation
  minus full messageHistory — truncate to last 5 messages; include turnLog
  truncated to last 3 entries; include memoryItemCount as int). Table updater
  consumes ConversationEntity events. ONE query getAllConversations SELECT * AS
  conversations FROM conversation_view. No WHERE status filter — caller filters
  client-side (Lesson 2).

- 1 Consumer MessageQueueConsumer subscribed to MessageQueue events; on
  MessageSubmitted starts or resumes a ConversationWorkflow turn with
  conversationId as the workflow id.

- 2 TimedActions:
  * SessionSimulator — every 90s, reads next line from
    src/main/resources/sample-events/session-prompts.jsonl and calls
    MessageQueue.enqueueMessage.
  * StuckConversationMonitor — every 30s, queries ConversationView.getAllConversations,
    filters PROCESSING conversations whose createdAt is older than 4 minutes,
    calls ConversationEntity.timeoutFail.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with POST /conversations, GET /conversations
    (filters client-side), GET /conversations/{id}, GET /conversations/sse,
    GET /conversations/{id}/memory,
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- RouterTasks.java declaring two Task<R> constants: CLASSIFY_TURN
  (resultConformsTo TurnPlan), REVISE_TURN (resultConformsTo TurnPlan).
- MemoryTasks.java declaring two Task<R> constants: RECALL
  (resultConformsTo MemoryContext), EXTRACT_MEMORIES (resultConformsTo MemoryDelta).
- ToolTasks.java declaring four Task<R> constants: EVALUATE, LOOK_UP,
  WEB_FETCH, RUN_EXPRESSION (all resultConformsTo ToolResult).
- Domain records as listed in SPEC §5.
- application/PiiScrubber.java — deterministic regex scrubber.
  Patterns: email (RFC 5322 simplified), E.164 phone (+[\d]{7,15}),
  US SSN (\d{3}-\d{2}-\d{4}), UK NINO ([A-Z]{2}\d{6}[A-D]),
  IPv4 (\d{1,3}(?:\.\d{1,3}){3}), IPv6 (simplified).
  Replacements: [REDACTED:email], [REDACTED:phone], [REDACTED:national-id],
  [REDACTED:ip-address].
- application/ToolDispatchGuardrail.java — deterministic vetter.
  Reject if the tool is not in (CALCULATOR, KNOWLEDGE_BASE, WEB_LOOKUP,
  CODE_RUNNER, NONE), if a CODE_RUNNER query matches
  /(os\.|subprocess\.|__import__|exec\(|eval\(|open\()/, if a WEB_LOOKUP
  query names a host not on the allow-list (akka.io, doc.akka.io, github.com,
  wikipedia.org), if a CALCULATOR expression exceeds 512 characters.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9428 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/session-prompts.jsonl with 8 canned
  conversation turns spanning calculator, knowledge base, web lookup, and
  code runner needs.
- src/main/resources/sample-data/knowledge-fixtures.jsonl — 12 canned
  knowledge facts (topic, question, answer). Used by KnowledgeBaseAgent.
- src/main/resources/sample-data/web-snippets.jsonl — 8 canned web snippets
  (host, title, excerpt). Used by WebLookupAgent.
- src/main/resources/sample-data/calc-fixtures.jsonl — 10 canned expression/result
  pairs. Used by CalculatorAgent.
- src/main/resources/sample-data/code-fixtures.jsonl — allow-listed expression
  set with canned outputs. Include one fixture whose output contains a raw
  IPv4 address so the J3-variant sanitizer test fires on memory extraction.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies for the metadata endpoint to serve from classpath).
- eval-matrix.yaml at the project root with 2 controls (G1, S1) and a
  matching simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations, and compliance sections;
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router.md, prompts/memory.md, prompts/calculator.md,
  prompts/knowledge-base.md, prompts/web-lookup.md, prompts/code-runner.md
  loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: AI Chatbot with
  Long-Term Memory and Dynamic Tool Routing", one-line pitch, prerequisites
  (including the integration form's host-software requirement: None),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO
  governance-mechanisms section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (conversation form + operator halt/resume control + live
  conversation list with status pills and expand-on-click for message
  history, turn log, and memory store). Browser title exactly:
  <title>Akka Sample: AI Chatbot with Long-Term Memory and Dynamic Tool
  Routing</title>. No subtitle on the Overview tab.

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
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://.
    (e) Type once in this session — value lives in Claude session memory only.
- NEVER write the key value to any file.

Mock LLM provider — required when option (a) is selected:
- Generate MockModelProvider.java dispatching on agent class name and Task<R> id.
  Per-agent mock-response files:
    router.json — two lists keyed by task id:
      "CLASSIFY_TURN" → 6 TurnPlan entries (spanning all four ToolKind values
      plus NONE, with plausible toolQuery strings and 1-2 MemoryItem updates).
      "REVISE_TURN" → 3 TurnPlan entries with tool=NONE and a direct-reply
      MemoryItem.
    memory.json — two lists:
      "RECALL" → 5 MemoryContext entries (0–4 relevantFacts).
      "EXTRACT_MEMORIES" → 5 MemoryDelta entries including at least one whose
      toAdd list contains a raw email address (alice@example.com) so the J3
      sanitizer test fires.
    calculator.json — 6 ToolResult entries (ok=true, content = numeric results).
    knowledge-base.json — 6 ToolResult entries with 3–5 line factual answers.
    web-lookup.json — 5 ToolResult entries with short snippet-based answers.
    code-runner.json — 5 ToolResult entries; one entry's content contains a
      raw IPv4 address (192.168.0.1) so the memory-level PII sanitizer test
      can be exercised.
- MockModelProvider.seedFor(conversationId) makes selection deterministic per
  conversation so the same session in dev reproduces the same output.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent.
- Lesson 4: WorkflowSettings.stepTimeout set explicitly on every step that
  calls an agent (recallStep, routeStep, executeStep, extractMemoriesStep,
  replyStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and on
  the Conversation entity state (turnLog, failureReason, haltReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion RouterTasks.java, MemoryTasks.java,
  and ToolTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current lineup.
  Conservative defaults: claude-sonnet-4-6, gpt-4o, gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never "mvn akka:run".
- Lesson 10: HTTP port 9428 in application.conf.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS overrides
  AND themeVariables.
- Lesson 25: API-key sourcing follows the five-option flow above.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute, NEVER
  by NodeList index. No "hidden" zombie panels in the DOM.
- The Overview tab's Try-it card shows just "/akka:build".
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars; the key-sourcing flow written into Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
