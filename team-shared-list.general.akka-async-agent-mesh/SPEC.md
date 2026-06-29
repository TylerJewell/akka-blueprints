# SPEC — akka-async-agent-mesh

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Async Multi-Agent Communication.
**One-line pitch:** Two independent agents, each with their own long-term memory, exchange fire-and-forget messages through a `send_message_to_agent_async` tool — neither blocks waiting for a reply, and a before-tool-call guardrail gates every inter-agent send before it executes.

## 2. What this blueprint demonstrates

The **team-shared-list** coordination pattern applied to asynchronous inter-agent messaging: a dispatcher agent composes an outbound message and calls `send_message_to_agent_async`, which records the message on a shared `MessageBusView` and returns a delivery receipt immediately — the dispatcher does not pause, it continues with its next task while the receiving agent processes the message on its own schedule. A processor agent runs in a separate durable `MessageProcessingWorkflow`, reads incoming messages off the shared list, applies them against its own long-term `AgentMemoryEntity`, and may enqueue follow-up messages back. The pattern also demonstrates two governance mechanisms appropriate for cross-agent communication: a **before-tool-call guardrail** that refuses sends to unknown or disallowed agent identities, and a **PII sanitizer** that scrubs personal data from message payloads before they are stored in any entity or view.

## 3. User-facing flows

The user opens the App UI tab, selects a scenario from the form (a topic and an initiating agent), and triggers a dispatch.

1. `MeshEndpoint` writes a `DispatchRequest` to `MessageBusEntity` (single instance "default") and starts a `MessageDispatchWorkflow` for the request id.
2. `MessageDispatchWorkflow` runs `DispatcherAgent`, which composes a `MessagePayload` and calls the `send_message_to_agent_async` tool. The tool call passes through the before-tool-call guardrail; if the recipient is not in the allowed roster the send is refused and the request records `SEND_BLOCKED`.
3. On approval, the tool records a `MessageEntity` (status `DISPATCHED`) and emits `MessageDispatched`. The workflow stores the `DeliveryReceiptEntity` and terminates — the dispatcher is done.
4. `MessageBusConsumer` subscribes to `MessageDispatched` and starts a `MessageProcessingWorkflow` for the message id.
5. `MessageProcessingWorkflow` runs `ProcessorAgent`. Before writing any output the PII sanitizer checks the message payload; any detected PII is replaced with a redaction token. The processor reads relevant entries from its `AgentMemoryEntity`, produces a `ProcessingResult`, and may enqueue one follow-up `DispatchRequest` (triggering a new dispatch cycle).
6. The workflow writes the `ProcessingResult` to `MessageEntity` (status → `PROCESSED`) and updates `AgentMemoryEntity` with new memory entries derived from the exchange.
7. `MessageBusView` projects every `MessageEntity` transition; the App UI streams updates via SSE.

A `ScenarioSimulator` (TimedAction) drips a sample dispatch scenario every 60 seconds. A `StaleMessageMonitor` (TimedAction) flags messages still in `DISPATCHED` after three minutes as `STALE`, allowing the UI to surface them.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `DispatcherAgent` | `AutonomousAgent` | Composes an outbound `MessagePayload` and calls `send_message_to_agent_async`. A before-tool-call guardrail gates every send. | `MessageDispatchWorkflow` | returns typed `DispatchResult` to workflow |
| `ProcessorAgent` | `AutonomousAgent` | Processes an inbound message against the agent's long-term memory; returns a `ProcessingResult`; may enqueue a follow-up dispatch. | `MessageProcessingWorkflow` | returns typed `ProcessingResult` to workflow |
| `MessageDispatchWorkflow` | `Workflow` | Runs `DispatcherAgent`, records `MessageEntity`, stores `DeliveryReceiptEntity`. | `MessageBusConsumer` (for follow-ups) + `MeshEndpoint` | `DispatcherAgent`, `MessageEntity`, `DeliveryReceiptEntity` |
| `MessageProcessingWorkflow` | `Workflow` | Per-message processing loop: receive → sanitize → process → memory update → optional follow-up. | `MessageBusConsumer` | `ProcessorAgent`, `MessageEntity`, `AgentMemoryEntity` |
| `MessageEntity` | `EventSourcedEntity` | One per message. Tracks delivery and processing lifecycle. | `MessageDispatchWorkflow`, `MessageProcessingWorkflow`, `StaleMessageMonitor` | `MessageBusView` |
| `AgentMemoryEntity` | `EventSourcedEntity` | One per agent id. Holds durable long-term memory entries (key-tagged facts). | `MessageProcessingWorkflow` | `MeshEndpoint` (read), `ProcessorAgent` (context) |
| `DeliveryReceiptEntity` | `EventSourcedEntity` | One per message. Stores the async delivery acknowledgement. | `MessageDispatchWorkflow` | `MeshEndpoint` (read) |
| `MessageBusEntity` | `EventSourcedEntity` | Single instance "default". Logs each `DispatchRequest` for audit/replay. | `MeshEndpoint`, `ScenarioSimulator` | `MessageBusConsumer` |
| `SystemControl` | `KeyValueEntity` | Operator halt flag for the whole mesh. | `MeshEndpoint` | `MessageDispatchWorkflow`, `DispatcherAgent` guardrail |
| `MessageBusView` | `View` | Shared message list. One row per `MessageEntity`. Read by the UI and the processing workflows. | `MessageEntity` events | `MessageProcessingWorkflow`, `MeshEndpoint` |
| `MessageBusConsumer` | `Consumer` | Subscribes to `MessageBusEntity` events; starts a `MessageDispatchWorkflow` per `DispatchRequest`. | `MessageBusEntity` events | `MessageDispatchWorkflow` |
| `ScenarioSimulator` | `TimedAction` | Drips a sample dispatch scenario every 60 s. | scheduler | `MessageBusEntity` |
| `StaleMessageMonitor` | `TimedAction` | Every 90 s, flags `DISPATCHED` messages older than 3 minutes as `STALE`. | scheduler | `MessageEntity` |
| `MeshEndpoint` | `HttpEndpoint` | `/api/*` — trigger dispatch, list messages, get memory, halt/resume, SSE. | — | `MessageBusEntity`, `MessageBusView`, `AgentMemoryEntity`, `DeliveryReceiptEntity`, `SystemControl` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules the two TimedActions; starts `MessageDispatchWorkflow` housekeeping. | — | scheduler |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record DispatchRequest(String requestId, String fromAgent, String toAgent,
                       String topic, String body, Instant requestedAt) {}

record MessagePayload(String messageId, String fromAgent, String toAgent,
                      String topic, String body, Instant composedAt) {}

record DispatchResult(String messageId, String deliveryReceiptId,
                      Optional<String> blockedReason) {}

record MemoryEntry(String entryId, String agentId, String tag,
                   String content, Instant recordedAt) {}

record ProcessingResult(String messageId, String processorAgent,
                        String summary, List<MemoryEntry> newMemory,
                        Optional<DispatchRequest> followUp) {}

record DeliveryReceipt(String receiptId, String messageId,
                       String fromAgent, String toAgent, Instant acknowledgedAt) {}
```

### Entity state — `Message` (state of `MessageEntity`, basis of the view row)

```java
record Message(
    String messageId,
    String requestId,
    String fromAgent,
    String toAgent,
    String topic,
    String body,
    MessageStatus status,
    Optional<String>  deliveryReceiptId,
    Optional<Instant> deliveredAt,
    Optional<String>  processingSummary,
    Optional<String>  blockedReason,
    Optional<Instant> processedAt,
    Optional<Instant> staledAt,
    Instant createdAt
) {}

enum MessageStatus { DISPATCHED, DELIVERED, PROCESSING, PROCESSED, SEND_BLOCKED, STALE }
```

### Entity state — `AgentMemory` (state of `AgentMemoryEntity`)

```java
record AgentMemory(
    String agentId,
    List<MemoryEntry> entries,
    Optional<Instant> lastUpdatedAt
) {}
```

### Events

`MessageEntity`: `MessageDispatched`, `MessageDelivered`, `MessageProcessingStarted`, `MessageProcessed`, `MessageSendBlocked`, `MessageStaled`.
`AgentMemoryEntity`: `MemoryEntryAdded`, `MemoryCleared`.
`DeliveryReceiptEntity`: `ReceiptRecorded`.
`MessageBusEntity`: `DispatchRequestSubmitted`.
`SystemControl` is a `KeyValueEntity` — state transitions are command-driven, no events.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/dispatch` — body `{ fromAgent, toAgent, topic, body }` → `{ requestId }`. Logs the request and starts dispatch.
- `GET /api/messages` — list all messages. Optional `?fromAgent=...`, `?toAgent=...`, `?status=...`, filtered client-side.
- `GET /api/messages/{id}` — one message with full lifecycle fields.
- `GET /api/messages/sse` — server-sent events stream of every message status change.
- `GET /api/agents/{agentId}/memory` — memory entries for an agent.
- `GET /api/receipts/{messageId}` — delivery receipt for a message.
- `POST /api/control/halt` — body `{ reason, by }`. Sets the operator halt flag.
- `POST /api/control/resume` — clears the halt flag.
- `GET /api/control` — current halt state.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Async Multi-Agent Communication</title>`.

- **Overview** — eyebrow "Overview" + headline "Async Multi-Agent Communication"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, message state machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a dispatch trigger form, a live message board grouped by status column (Dispatched / Delivered / Processing / Processed / Send Blocked / Stale), per-message cards showing sender/recipient agents and the processing summary, a per-agent memory panel, and a Halt / Resume control.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`tool-permission` flavor, on `DispatcherAgent`): every call to `send_message_to_agent_async` passes through a guardrail that refuses the send when the recipient agent id is not in the allowed roster, when the message body exceeds the size limit, or when `SystemControl` reports the mesh is halted. A refused send records the request as `SEND_BLOCKED` with the guardrail reason. Blocking.
- **S1 — PII sanitizer** (`pii` flavor, in `MessageProcessingWorkflow` before any entity write): message payloads are scanned for PII tokens (email addresses, phone patterns, national ID patterns) before being written to `AgentMemoryEntity` or `MessageBusView`; detected values are replaced with `[REDACTED:<type>]`. Non-blocking at the workflow level — the message proceeds with the sanitized payload.

## 9. Agent prompts

- `DispatcherAgent` → `prompts/dispatcher.md`. Composes a `MessagePayload` and calls `send_message_to_agent_async`.
- `ProcessorAgent` → `prompts/processor.md`. Processes an inbound message against the agent's memory; may enqueue a follow-up.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Trigger a dispatch; dispatcher composes and sends a message; processor receives, processes, and writes a result; message reaches `PROCESSED`. Board updates live via SSE.
2. **J2** — Dispatcher fires and returns a receipt immediately; processor runs asynchronously; both agents progress in parallel without either blocking the other.
3. **J3** — Message payload contains PII; sanitizer redacts it before any entity write; original value never appears in `AgentMemoryEntity` or `MessageBusView`.
4. **J4** — Dispatcher addresses a message to an unknown agent id; guardrail refuses the send; request is recorded `SEND_BLOCKED` with the refusal reason; no `MessageEntity` is created.
5. **J5** — Operator halts the mesh; no new dispatches start; resume restores normal flow.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named akka-async-agent-mesh demonstrating the
team-shared-list × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact team-shared-list-general-akka-async-agent-mesh.
Java package io.akka.samples.asyncmultiagentcommunication. Akka 3.6.0. HTTP port 9497.

Components to wire (exactly):
- 2 AutonomousAgents:
  * DispatcherAgent — definition() with capability(MeshTasks.of(DISPATCH)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/dispatcher.md.
    Returns DispatchResult{messageId, deliveryReceiptId, blockedReason: Optional}.
    Register a before-tool-call guardrail on this agent (control G1).
  * ProcessorAgent — definition() with capability(MeshTasks.of(PROCESS)
    .maxIterationsPerTask(3)). System prompt from prompts/processor.md.
    Returns ProcessingResult{messageId, processorAgent, summary,
    newMemory: List<MemoryEntry>, followUp: Optional<DispatchRequest>}.
    Run as two runtime instances addressed by instanceId (agent-alpha, agent-beta) —
    ONE agent class, two instance ids; never generate two agent classes.

- 1 Workflow MessageDispatchWorkflow with steps composeStep -> sendStep ->
  receiptStep. composeStep calls forAutonomousAgent(DispatcherAgent.class, fromAgent)
  .runSingleTask(DISPATCH.instructions(request)) then forTask(requestId).result(DISPATCH).
  sendStep: if the DispatchResult has a blockedReason, call
  MessageBusEntity.recordBlocked(requestId, reason) and terminate. Otherwise create
  MessageEntity (status DISPATCHED) and emit MessageDispatched. receiptStep: call
  DeliveryReceiptEntity.record(receipt) and end. Override settings() with
  stepTimeout(composeStep, 90s).

- 1 Workflow MessageProcessingWorkflow, one instance per messageId. Steps:
  receiveStep -> sanitizeStep -> processStep -> memoryStep -> followUpStep. receiveStep:
  read SystemControl; if halted, schedule a 5s resume and pause. Otherwise load
  MessageEntity and AgentMemoryEntity(toAgent) context and go to sanitizeStep.
  sanitizeStep: run PiiSanitizer.sanitize(payload); update local state with
  sanitizedBody; call MessageEntity.startProcessing. processStep (stepTimeout 90s):
  call forAutonomousAgent(ProcessorAgent.class, toAgent)
  .runSingleTask(PROCESS.instructions(message, memory)). memoryStep: call
  AgentMemoryEntity(toAgent).addEntries(result.newMemory) and
  MessageEntity.complete(result.summary) (status -> PROCESSED). followUpStep: if
  result.followUp is present, call MessageBusEntity.submitDispatch(followUp) and
  terminate; else terminate.

- 4 EventSourcedEntities:
  * MessageEntity holding Message state. Commands: createMessage, deliver,
    startProcessing, complete, blockSend, stale, getMessage. MessageStatus enum:
    DISPATCHED, DELIVERED, PROCESSING, PROCESSED, SEND_BLOCKED, STALE. Events:
    MessageDispatched, MessageDelivered, MessageProcessingStarted, MessageProcessed,
    MessageSendBlocked, MessageStaled. emptyState() returns Message.initial("", "")
    with NO commandContext() reference.
  * AgentMemoryEntity, id = agentId. Commands: addEntries(List<MemoryEntry>),
    clearMemory, getMemory. Events: MemoryEntryAdded, MemoryCleared. State is
    AgentMemory{agentId, entries, lastUpdatedAt: Optional}.
    emptyState() returns AgentMemory.initial("") with NO commandContext() reference.
  * DeliveryReceiptEntity, id = messageId. Command record(DeliveryReceipt) emitting
    ReceiptRecorded{receipt}. State is Optional<DeliveryReceipt>.
  * MessageBusEntity, single instance "default". Command submitDispatch(DispatchRequest)
    emitting DispatchRequestSubmitted{requestId, fromAgent, toAgent, topic, body,
    requestedAt}. Also command recordBlocked(requestId, reason) emitting
    DispatchRequestBlocked{requestId, reason, blockedAt}.

- 1 KeyValueEntity SystemControl, single instance "default", holding
  {halted: boolean, haltedReason: Optional<String>, haltedBy: Optional<String>,
  haltedAt: Optional<Instant>}. Commands: halt(reason, by), resume, getControl.

- 1 View MessageBusView with row type MessageRow (mirrors Message minus raw body
  content for SSE efficiency — keeps topic, processingSummary, blockedReason only).
  Table updater consumes MessageEntity events. ONE query getAllMessages SELECT * AS
  messages FROM message_bus. No WHERE status filter (Lesson 2); callers filter
  client-side.

- 1 Consumer MessageBusConsumer subscribed to MessageBusEntity events; on
  DispatchRequestSubmitted starts a MessageDispatchWorkflow with requestId as the
  workflow id.

- 2 TimedActions:
  * ScenarioSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/dispatch-scenarios.jsonl and calls
    MessageBusEntity.submitDispatch. Stops cycling when the file is exhausted (wraps).
  * StaleMessageMonitor — every 90s, queries MessageBusView.getAllMessages, finds
    messages in DISPATCHED whose createdAt is older than 3 minutes, and calls
    MessageEntity.stale.

- 2 HttpEndpoints:
  * MeshEndpoint at /api with POST /dispatch, GET /messages, GET /messages/{id},
    GET /messages/sse, GET /agents/{agentId}/memory, GET /receipts/{messageId},
    POST /control/halt, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

- 1 service-setup Bootstrap: schedule ScenarioSimulator and StaleMessageMonitor.

Companion files:
- MeshTasks.java declaring two Task<R> constants: DISPATCH (resultConformsTo
  DispatchResult) and PROCESS (resultConformsTo ProcessingResult).
- PiiSanitizer.java (deterministic, in application/) — a pure function over a String
  returning a sanitized String: replaces patterns matching email addresses
  ([\\w.+]+@[\\w.]+\\.[a-z]{2,}), phone numbers (\\b\\d{3}[-.\\s]?\\d{3}[-.\\s]?\\d{4}\\b),
  and 9-digit national IDs (\\b\\d{3}-\\d{2}-\\d{4}\\b) with [REDACTED:EMAIL],
  [REDACTED:PHONE], [REDACTED:NATIONAL_ID] respectively. NOT an LLM call. This is
  the sanitizer (S1).
- Domain records DispatchRequest, MessagePayload, DispatchResult, MemoryEntry,
  ProcessingResult, DeliveryReceipt in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9497
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also an
  agent-roster setting async-mesh.agents = ["agent-alpha","agent-beta"] read by
  the G1 guardrail to validate recipient ids.
- src/main/resources/sample-events/dispatch-scenarios.jsonl with 6 canned dispatch
  scenarios (each a DispatchRequest JSON object — e.g., agent-alpha asks agent-beta
  about a known fact, agent-beta sends a status report, a cross-topic query with a
  follow-up, a message containing PII to exercise S1, a message to an unknown
  recipient to exercise G1, a normal round-trip).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 2 controls (G1 before-tool-call
  guardrail, S1 pii sanitizer) and a matching simplified_view list.
  No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data classes,
  capabilities, model family, and oversight; deployer-specific fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/dispatcher.md and prompts/processor.md loaded at agent startup as system
  prompts.
- README.md at the project root: title "Akka Sample: Async Multi-Agent Communication",
  one-line pitch, prerequisites (host software: none), generate-the-system,
  what-you-get, customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual" prefix on
  tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity
  0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows and a colored mechanism pill in the ID column), App
  UI (dispatch form + a board with one column per MessageStatus, a per-agent
  memory panel, and a Halt/Resume control). Browser title exactly:
  <title>Akka Sample: Async Multi-Agent Communication</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one
  is set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in .akka-build.yaml.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml.
    (e) Type once in this session — value lives in Claude session memory; gone
        when the session ends.
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the
  ModelProvider interface with per-agent dispatch on agent class name / Task<R> id.
- Per-agent mock-response shapes for THIS blueprint:
    dispatcher.json — 4-6 DispatchResult entries. Most have a messageId, a
      deliveryReceiptId, and an empty blockedReason. Include 1 entry with a
      non-empty blockedReason (unknown recipient) to exercise the G1 journey.
    processor.json — 4-6 ProcessingResult entries. Most have a summary,
      1-2 MemoryEntry items with believable key-tagged facts, and an empty followUp.
      Include 1 entry whose followUp is present (toAgent "agent-alpha", a concrete
      follow-up topic) to exercise the async-chained dispatch journey. Include 1
      entry whose incoming body contains a PII pattern (an email address) to
      exercise the S1 sanitizer journey.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion MeshTasks Task<R> constants (Lesson 7).
- View has no WHERE filter on the enum status column; filter client-side (Lesson 2).
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9497 (Lessons 10, 13).
- The generated static-resources/index.html must include the mermaid CSS overrides
  AND theme variables from Lesson 24 (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc). Without these,
  state names render black-on-black and arrow labels clip.
- Tab switching in static-resources/index.html MUST match by data-tab / data-panel
  attribute, NEVER by NodeList index (Lesson 26). No "hidden" zombie panels in the
  DOM — delete removed tabs, do not display:none them.
- UI is a single self-contained static-resources/index.html — no ui/ folder, no
  package.json, no npm build (Lesson 17).
- The Overview tab's Try-it card shows just "/akka:build" — not an env-var export
  block. Per Lesson 25, /akka:specify handles the key during generation.
- No forbidden words in user-facing text (Lessons 21, 22, 23).
- Lessons 1, 4, 6, 7, 8, 9, 10, 11, 12, 13, 23, 24, 25, 26 apply.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key (offer the three valid env vars plus the mock-LLM option from Section 11), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
