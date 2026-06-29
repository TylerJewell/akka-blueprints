# SPEC — per-tenant-agent-fleet

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Per-Customer Success CRM Agents.
**One-line pitch:** Register a customer; a control-plane agent spawns one dedicated stateful agent per customer that researches their background, sends a personalized welcome email, maintains interaction memory, and continues supporting them through an ongoing chat — the whole fleet monitored at runtime and measured by a periodic quality eval.

## 2. What this blueprint demonstrates

The **composite-multi-team** coordination pattern wired with Akka's first-party primitives: a top-level `FleetCoordinator` control plane delegates to a fleet of per-customer agents, each of which runs a different internal coordination capability. The **onboarding stage** uses **delegation**: the `AgentFleetWorkflow` fans out a `BackgroundResearcher` instance per customer to build a profile, then uses the result to call the outbound email tool through `CustomerTools`. The **ongoing-chat stage** uses a **team over a shared memory**: the `ChatResponder` reads from and writes to the `AgentMemoryEntity` so each response is informed by prior interaction. The **fleet governance** uses **moderation**: `FleetEvalConsumer` samples active agents and aggregates scores across the fleet into a fleet health grade.

The blueprint also demonstrates four governance mechanisms a CRM fleet needs: a **PII sanitizer** on every inbound customer record and chat message, a **before-tool-call guardrail** on the outbound email tool, a **deployer-runtime monitor** for fleet-wide health, and a **periodic performance eval** that measures welcome and chat quality continuously.

## 3. User-facing flows

The user opens the App UI tab and registers a customer via the form (a customer name, email, and CRM tier).

1. The system logs the registration on `CustomerRegistry` and a `Consumer` starts an `AgentFleetWorkflow` for the customer.
2. The PII sanitizer strips any sensitive fields from the input before the workflow begins.
3. The `FleetCoordinator` agent registers the new customer and records a fleet entry (`FleetEntry` status `SPAWNING`).
4. **Onboarding stage (delegation).** The `BackgroundResearcher` receives the CRM record and produces a `CustomerProfile` (account tier, key facts, interaction hints). The workflow stores the profile in `AgentMemoryEntity`. The fleet entry moves to `ONBOARDING`.
5. **Welcome stage (tool call with guardrail).** The `AgentFleetWorkflow` calls `CustomerTools.sendWelcomeEmail` with the profile's personalized greeting. A before-tool-call guardrail vets the call: it refuses a disallowed recipient domain, an oversized body, or a template that exceeds the configured tone threshold. On a refusal the workflow records the block reason in memory and continues to `READY` without sending. On success the fleet entry moves from `ONBOARDING` to `READY` and a `WelcomeSent` event is recorded.
6. **Ongoing chat (team over shared memory).** While the agent is `READY`, any `ChatMessage` submitted for the customer triggers the `ChatResponder` agent. The responder reads the customer's full `AgentMemory` (profile + prior turns), generates a reply, and writes the new turn back into `AgentMemoryEntity`. The fleet entry stays `READY`; each turn appends to the `ChatHistoryView`.
7. A `FleetEvalConsumer` fires a non-blocking periodic eval on `WelcomeSent` and `ChatTurnCompleted` events, recording a `FleetEval` (stage, score, flags) against the customer's memory — surfaced in the fleet dashboard.
8. The `StaleAgentMonitor` (TimedAction) scans the `FleetStatusView` every 5 minutes and emits a `FleetHealthSummary` for any customer whose last interaction is older than the configured window (default 7 days), giving the deployer visibility into dormant agents.
9. A `CustomerSimulator` (TimedAction) drips a sample customer registration every 60 s so the App UI is not empty when first loaded.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `FleetCoordinator` | `AutonomousAgent` | Control-plane agent: registers a new customer into the fleet, records a `FleetEntry`, and under a periodic eval aggregates fleet health. | `AgentFleetWorkflow`, `FleetEvalConsumer` | `CustomerRegistry`, returns `FleetEntry` |
| `BackgroundResearcher` | `AutonomousAgent` | Produces a `CustomerProfile` from a CRM record. One instance per customer. | `AgentFleetWorkflow` | returns `CustomerProfile` |
| `ChatResponder` | `AutonomousAgent` | Handles one chat turn for a given customer; reads and writes `AgentMemoryEntity`. | `AgentFleetWorkflow` | `CustomerTools` → `AgentMemoryEntity`, returns `ChatReply` |
| `AgentFleetWorkflow` | `Workflow` | Per-customer durable pipeline: register → research → welcome → ready. Also handles each incoming chat message step. | `CustomerIngestConsumer` | all agents, `CustomerRegistry`, `AgentMemoryEntity` |
| `CustomerRegistry` | `EventSourcedEntity` | One-per-customer record: name, email, tier, status, profile. Commands: `registerCustomer`, `updateProfile`, `markReady`, `getCustomer`. | `AgentFleetWorkflow`, `FleetEndpoint`, `CustomerSimulator` | `FleetStatusView`, `CustomerIngestConsumer` |
| `AgentMemoryEntity` | `EventSourcedEntity` | Per-customer interaction memory: profile, chat turns, eval results, email-send record. Commands: `initMemory`, `storeProfile`, `recordWelcome`, `recordWelcomeBlock`, `appendChatTurn`, `recordFleetEval`, `getMemory`. | `AgentFleetWorkflow`, `CustomerTools`, `FleetEvalConsumer` | `ChatHistoryView` |
| `FleetStatusView` | `View` | Fleet overview read model: one row per customer with status, tier, profile summary, last-interaction time, and latest eval. | `CustomerRegistry` events, `AgentMemoryEntity` events | `FleetEndpoint`, `StaleAgentMonitor` |
| `ChatHistoryView` | `View` | Per-customer chat-history read model: ordered list of `ChatRow` records. | `AgentMemoryEntity` events | `FleetEndpoint` |
| `CustomerIngestConsumer` | `Consumer` | Subscribes to `CustomerRegistry` events; on `CustomerRegistered` starts an `AgentFleetWorkflow` for that customer. | `CustomerRegistry` events | `AgentFleetWorkflow` |
| `FleetEvalConsumer` | `Consumer` | Subscribes to `AgentMemoryEntity` events; on `WelcomeSent` and `ChatTurnCompleted` runs a deterministic `FleetEvaluator` and records a `FleetEval`. | `AgentMemoryEntity` events | `AgentMemoryEntity` |
| `CustomerSimulator` | `TimedAction` | Every 60 s, registers the next sample customer from `sample-events/customers.jsonl`. | scheduler | `CustomerRegistry` |
| `StaleAgentMonitor` | `TimedAction` | Every 5 min, queries `FleetStatusView` and records a `FleetHealthSummary` for any customer idle > 7 days. | scheduler | `AgentMemoryEntity` |
| `FleetEndpoint` | `HttpEndpoint` | `/api/*` — register, get customer, list customers, chat, get chat history, SSE, fleet health, metadata. | — | `CustomerRegistry`, `FleetStatusView`, `ChatHistoryView`, `AgentMemoryEntity`, `AgentFleetWorkflow` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |
| `Bootstrap` | service-setup | Schedules `CustomerSimulator` and `StaleAgentMonitor`; pre-warms the `FleetCoordinator` instance. | — | `FleetCoordinator`, scheduler |

Companion classes that are not Akka components: `FleetTasks` (the `Task<R>` constants), `CustomerTools` (the function tool `ChatResponder` and `AgentFleetWorkflow` call for outbound operations; the before-tool-call guardrail vets every call), `PiiSanitizer` (a deterministic pure function that strips configured PII patterns from inbound strings before the workflow processes them), and `FleetEvaluator` (the deterministic pure function that produces `FleetEval` results without an LLM call).

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record CustomerBrief(String customerId, String name, String email, CrmTier tier) {}

record CustomerProfile(String customerId, CrmTier tier, String summary, List<String> keyFacts, List<String> interactionHints) {}

record WelcomeEmail(String recipientEmail, String subject, String body, String templateId) {}
record WelcomeSendResult(String customerId, boolean sent, Optional<String> blockReason) {}

record ChatMessage(String customerId, String messageId, String content, Instant receivedAt) {}
record ChatReply(String customerId, String messageId, String reply, Instant repliedAt) {}

record ChatTurn(String messageId, String customerContent, String agentReply, Instant turnAt) {}
record AgentMemory(String customerId, Optional<CustomerProfile> profile, Optional<WelcomeSendResult> welcomeResult, List<ChatTurn> turns, List<FleetEval> evals) {}

record FleetEval(String stage, int score, List<String> flags, Instant evaluatedAt) {}
record FleetEntry(String customerId, String name, CrmTier tier, FleetStatus status, Instant registeredAt) {}
record FleetHealthSummary(String customerId, long daysSinceLastInteraction, String recommendation) {}
```

### Entity state — `Customer` (state of `CustomerRegistry`, basis of the fleet status row)

```java
record Customer(
    String customerId,
    String name,
    String email,
    CrmTier tier,
    CustomerStatus status,
    Optional<CustomerProfile> profile,
    Optional<Instant>         profiledAt,
    Optional<Instant>         readyAt,
    Instant                   registeredAt
) {}

enum CustomerStatus { REGISTERED, ONBOARDING, READY }
enum CrmTier { STANDARD, PREMIUM, ENTERPRISE }
```

### Entity state — `AgentMemory` (state of `AgentMemoryEntity`)

```java
record AgentMemory(
    String                        customerId,
    Optional<CustomerProfile>     profile,
    Optional<WelcomeSendResult>   welcomeResult,
    List<ChatTurn>                turns,
    List<FleetEval>               evals,
    Optional<Instant>             lastInteractionAt
) {}
```

### Enums

```java
enum FleetStatus { SPAWNING, ONBOARDING, READY, DORMANT }
enum CustomerStatus { REGISTERED, ONBOARDING, READY }
enum CrmTier { STANDARD, PREMIUM, ENTERPRISE }
```

### Events

`CustomerRegistry`: `CustomerRegistered`, `ProfileUpdated`, `CustomerReadied`.
`AgentMemoryEntity`: `MemoryInitialized`, `ProfileStored`, `WelcomeRecorded`, `WelcomeBlockRecorded`, `ChatTurnAppended`, `FleetEvalRecorded`.

See `reference/data-model.md` for the full field-by-field table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/customers` — body `{ name, email, tier? }` → `202 { customerId }`. Registers a new customer and starts the onboarding pipeline.
- `GET /api/customers` — list all customers. Optional `?status=...`, filtered client-side.
- `GET /api/customers/{id}` — one customer with profile, welcome result, and latest eval.
- `GET /api/customers/sse` — server-sent events stream of every customer status change.
- `POST /api/customers/{id}/chat` — body `{ content }` → `200 { reply }`. Sends a chat message to the customer's agent; synchronous reply.
- `GET /api/customers/{id}/chat` — full chat history for the customer.
- `GET /api/customers/{id}/chat/sse` — SSE stream of new chat turns for one customer.
- `GET /api/fleet/health` — current `FleetHealthSummary` list from the latest monitor run.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Per-Customer Success CRM Agents</title>`.

- **Overview** — eyebrow "Overview" + headline "Per-Customer Success CRM Agents"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — the four mermaid diagrams (component graph, sequence, customer status machine, entity model) with the Akka theme variables and the Lesson 24 state-label CSS overrides, plus a click-to-expand component table with hand-tagged Java snippets.
- **Risk Survey** — the seven sections from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — a 5-column table with one row per control and click-to-expand rationale + implementation; the id badge carries a colored mechanism pill.
- **App UI** — a customer registration form; a live fleet board grouped by status with per-customer cards showing profile summary, welcome status, chat-turn count, latest eval score, and a chat input panel; plus a fleet-health summary panel updated live.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii` flavor, applied in `AgentFleetWorkflow` and `FleetEndpoint`): every inbound customer record and every chat message passes through `PiiSanitizer` — a deterministic pure function that strips or masks configured patterns (e.g., raw SSNs, card numbers) before the workflow or any agent sees the data. A sanitized payload replaces the original in memory; the raw value is never persisted. System-level, non-blocking.
- **G1 — before-tool-call guardrail** (`tool-permission` flavor, on `ChatResponder` and `AgentFleetWorkflow` when calling `CustomerTools`): every call to `sendWelcomeEmail` passes through a guardrail that refuses a disallowed recipient domain, a body exceeding the size cap, or a template marked restricted. A refused call records the block reason via `AgentMemoryEntity.recordWelcomeBlock` and the workflow continues without sending. Blocking.
- **HO1 — deployer-runtime monitoring** (`deployer-runtime-monitoring` flavor): `StaleAgentMonitor` runs every 5 minutes and builds a `FleetHealthSummary` for every customer whose last interaction exceeds the configured window. Summaries are returned by `GET /api/fleet/health` and shown in the App UI's health panel. The monitor never modifies agent state — it reports to the deployer so the deployer can decide. Non-blocking, system-level.
- **E1 — periodic performance eval** (`performance-monitor` flavor): `FleetEvalConsumer` subscribes to `AgentMemoryEntity` events and fires on `WelcomeRecorded` and `ChatTurnAppended`. It runs `FleetEvaluator` — a deterministic pure function, not an LLM call — that scores each interaction stage and records a `FleetEval`. Scores are aggregated in the fleet dashboard. Non-blocking.

## 9. Agent prompts

- `FleetCoordinator` → `prompts/fleet-coordinator.md`. Registers new customers and emits fleet health summaries.
- `BackgroundResearcher` → `prompts/background-researcher.md`. Builds a `CustomerProfile` from CRM data.
- `ChatResponder` → `prompts/chat-responder.md`. Handles one customer chat turn drawing on agent memory.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Register a customer; the background researcher builds a profile; the welcome email is dispatched under the guardrail; the agent enters `READY` and answers a chat message. The fleet board updates live via SSE.
2. **J2** — Two simultaneous customer registrations each produce exactly one dedicated agent with no cross-contamination of memory.
3. **J3** — The outbound email tool is called with a disallowed recipient; the before-tool-call guardrail blocks it; no email is sent and the block reason is recorded.
4. **J4** — A raw PII field appears in a chat message; the sanitizer strips it before the agent sees it.
5. **J5** — After the monitoring window elapses, the fleet health panel shows a summary for the dormant customer without changing their agent state.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named per-tenant-agent-fleet demonstrating the
composite-multi-team × cx-support cell. Runs out of the box (no external
services). Maven group io.akka.samples. Maven artifact
composite-multi-team-cx-support-akka-per-tenant-agent-fleet. Java package
io.akka.samples.percustomersuccesscrmagents. Akka 3.6.0. HTTP port 9512.

Components to wire (exactly):
- 3 AutonomousAgents (each extends akka.javasdk.agent.autonomous.AutonomousAgent;
  never silently downgrade to Agent):
  * FleetCoordinator — definition() with capability(TaskAcceptance.of(REGISTER_CUSTOMER)
    .maxIterationsPerTask(2)) and capability(TaskAcceptance.of(SUMMARIZE_FLEET)
    .maxIterationsPerTask(2)). System prompt loaded from prompts/fleet-coordinator.md.
    REGISTER_CUSTOMER returns FleetEntry{customerId, name, tier, status, registeredAt}.
    SUMMARIZE_FLEET returns List<FleetHealthSummary>.
  * BackgroundResearcher — capability(TaskAcceptance.of(RESEARCH_CUSTOMER)
    .maxIterationsPerTask(2)). System prompt from prompts/background-researcher.md.
    RESEARCH_CUSTOMER returns CustomerProfile{customerId, tier, summary, keyFacts,
    interactionHints}. Run as one instance per customerId (instanceId = customerId).
    ONE agent class, one instance per customer.
  * ChatResponder — capability(TaskAcceptance.of(RESPOND_TO_CHAT)
    .maxIterationsPerTask(3)). System prompt from prompts/chat-responder.md.
    RESPOND_TO_CHAT returns ChatReply{customerId, messageId, reply, repliedAt}.
    Run as one instance per customerId (instanceId = customerId). ONE agent class,
    one instance per customer. Equipped with CustomerTools; the G1 before-tool-call
    guardrail is registered on this agent.

- 1 Workflow AgentFleetWorkflow, one instance per customerId. State carries
  customerId, name, email, tier, profile (Optional), welcomeResult (Optional),
  and stage. Steps: registerStep -> researchStep -> welcomeStep -> readyStep.
  An additional chatStep handles each incoming chat message after readyStep.
  * registerStep (stepTimeout 30s): call forAutonomousAgent(FleetCoordinator.class,
    "fleet").runSingleTask(REGISTER_CUSTOMER.instructions(customerBrief)) then
    forTask(taskId).result(REGISTER_CUSTOMER); call
    CustomerRegistry.registerCustomer(customerBrief). Go to researchStep.
  * researchStep (stepTimeout 90s): call forAutonomousAgent(BackgroundResearcher.class,
    customerId).runSingleTask(RESEARCH_CUSTOMER.instructions(customerRecord)) then
    result(RESEARCH_CUSTOMER) -> CustomerProfile. Call
    AgentMemoryEntity.storeProfile(profile) and
    CustomerRegistry.updateProfile(profile). Go to welcomeStep.
  * welcomeStep (stepTimeout 60s): build a WelcomeEmail from the profile; call
    CustomerTools.sendWelcomeEmail (guarded by G1). If the guardrail refuses, call
    AgentMemoryEntity.recordWelcomeBlock(reason) and proceed to readyStep anyway.
    On success, call AgentMemoryEntity.recordWelcome(result) and
    CustomerRegistry.markReady. Go to readyStep.
  * readyStep: mark stage READY and self-suspend; chatStep resumes on receipt of
    each ChatMessage event routed to this workflow.
  * chatStep (stepTimeout 60s): call forAutonomousAgent(ChatResponder.class,
    customerId).runSingleTask(RESPOND_TO_CHAT.instructions(message, memory)) then
    result(RESPOND_TO_CHAT) -> ChatReply. Call
    AgentMemoryEntity.appendChatTurn(turn). Return reply to caller. Resume readyStep.
  Override settings() with stepTimeout(registerStep, 30s),
  stepTimeout(researchStep, 90s), stepTimeout(welcomeStep, 60s),
  stepTimeout(chatStep, 60s).

- 2 EventSourcedEntities:
  * CustomerRegistry holding Customer state, one per customerId. Commands:
    registerCustomer, updateProfile, markReady, getCustomer.
    CustomerStatus enum: REGISTERED, ONBOARDING, READY.
    CrmTier enum: STANDARD, PREMIUM, ENTERPRISE.
    Events: CustomerRegistered, ProfileUpdated, CustomerReadied.
    emptyState() returns Customer.initial("") with NO commandContext() reference.
  * AgentMemoryEntity holding AgentMemory state, one per customerId. Commands:
    initMemory, storeProfile, recordWelcome, recordWelcomeBlock, appendChatTurn,
    recordFleetEval, getMemory.
    Events: MemoryInitialized, ProfileStored, WelcomeRecorded, WelcomeBlockRecorded,
    ChatTurnAppended, FleetEvalRecorded.
    emptyState() returns AgentMemory.initial("") with NO commandContext() reference.

- 2 Views:
  * FleetStatusView with row type CustomerRow (mirrors Customer but adds
    lastInteractionAt, latestEvalScore). ONE query getAllCustomers SELECT * AS
    customers FROM fleet_status. No WHERE status filter (Lesson 2); callers filter
    client-side. Add a streamAllCustomers query for the SSE endpoint.
  * ChatHistoryView with row type ChatRow (customerId, messageId, customerContent,
    agentReply, turnAt). ONE query getChatHistory(customerId) SELECT * AS turns
    FROM chat_history WHERE customer_id = :customerId. Add a
    streamChatHistory(customerId) query for the SSE endpoint.

- 2 Consumers:
  * CustomerIngestConsumer subscribed to CustomerRegistry events; on
    CustomerRegistered calls AgentMemoryEntity.initMemory then starts an
    AgentFleetWorkflow with the customerId as the workflow id.
  * FleetEvalConsumer subscribed to AgentMemoryEntity events; on WelcomeRecorded
    and ChatTurnAppended it runs FleetEvaluator (pure function, NOT an LLM call)
    producing a FleetEval{stage, score, flags, evaluatedAt} and calls
    AgentMemoryEntity.recordFleetEval(eval). Ignore all other events. This is
    control E1.

- 2 TimedActions:
  * CustomerSimulator — every 60s, reads the next line from
    src/main/resources/sample-events/customers.jsonl (wraps when exhausted) and
    calls CustomerRegistry.registerCustomer.
  * StaleAgentMonitor — every 5 min, queries FleetStatusView.getAllCustomers,
    finds customers whose lastInteractionAt is older than 7 days (or is absent),
    and calls AgentMemoryEntity.recordFleetEval with a DORMANT-flag eval for each.
    Also emits a FleetHealthSummary list returned by GET /api/fleet/health.

- 2 HttpEndpoints:
  * FleetEndpoint at /api with POST /customers, GET /customers (client-side filter
    by status), GET /customers/{id}, GET /customers/sse, POST /customers/{id}/chat,
    GET /customers/{id}/chat, GET /customers/{id}/chat/sse, GET /fleet/health, and
    three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* -> static-resources/*.

- 1 service-setup Bootstrap: schedule CustomerSimulator and StaleAgentMonitor;
  initialize the FleetCoordinator instance "fleet".

Companion files:
- FleetTasks.java declaring the Task<R> constants: REGISTER_CUSTOMER (resultConformsTo
  FleetEntry), RESEARCH_CUSTOMER (CustomerProfile), RESPOND_TO_CHAT (ChatReply),
  SUMMARIZE_FLEET (List).
- CustomerTools.java — the function tool ChatResponder and AgentFleetWorkflow call
  for outbound operations: sendWelcomeEmail(customerId, email). Routes through
  AgentMemoryEntity for state updates. The G1 before-tool-call guardrail vets every
  call: refuse when the recipientEmail domain is in the blocked list, when the body
  exceeds the size cap, or when the templateId is marked restricted.
- PiiSanitizer.java (deterministic, in application/) — a pure function over a
  String that strips or masks configured patterns (SSN-like, card-number-like,
  raw phone numbers) before the workflow processes them. Applied at FleetEndpoint
  boundary and at the top of AgentFleetWorkflow before registerStep.
- FleetEvaluator.java (deterministic, in application/) — a pure function used by
  FleetEvalConsumer: scores a welcome or chat stage 0-100 and lists flags (e.g.,
  welcome block recorded, chat reply under a minimum length, profile missing key
  facts). NOT an LLM call.
- Domain records CustomerBrief, CustomerProfile, WelcomeEmail, WelcomeSendResult,
  ChatMessage, ChatReply, ChatTurn, AgentMemory, FleetEval, FleetEntry,
  FleetHealthSummary, and the enums CustomerStatus, CrmTier, FleetStatus in domain/.
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9512
  and akka.javasdk.agent model-provider blocks for anthropic (claude-sonnet-4-6),
  openai (gpt-4o), googleai-gemini (gemini-2.5-flash), each api-key read from
  ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. Also
  per-tenant-agent-fleet.stale-agent-days = 7 and
  per-tenant-agent-fleet.email-blocked-domains = ["example-blocked.com"] read by
  CustomerTools and StaleAgentMonitor.
- src/main/resources/sample-events/customers.jsonl with 6 canned customer records
  (each a self-contained CRM entry — e.g., Aiko Tanaka / PREMIUM, Miguel Ferreira /
  ENTERPRISE, Sara Okonkwo / STANDARD, Chen Wei / PREMIUM, Priya Nair / ENTERPRISE,
  Lukas Becker / STANDARD).
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md
  (copies of the root-level files for the endpoint to serve from classpath).
- eval-matrix.yaml at the project root with the 4 controls (S1 pii sanitizer,
  G1 before-tool-call guardrail, HO1 hotl deployer-runtime-monitoring, E1
  eval-periodic performance-monitor) and a matching simplified_view list. No
  regulation_anchors.
- risk-survey.yaml at the project root, pre-filling sector, decisions, data
  classes, capabilities, model family, and oversight; deployer-specific fields
  marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/fleet-coordinator.md, prompts/background-researcher.md,
  prompts/chat-responder.md loaded at agent startup as system prompts.
- README.md at the project root: title "Akka Sample: Per-Customer Success CRM
  Agents", one-line pitch, prerequisites (host software: none),
  generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section. NO "Visual" prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained HTML
  file (no ui/ folder, no npm build). Five tabs matching the formal exemplar:
  Overview, Architecture (4 mermaid diagrams + click-to-expand component table
  with syntax-highlighted Java snippets), Risk Survey (7 sections from
  governance.html with answers from risk-survey.yaml; unanswered .qb opacity
  0.45), Eval Matrix (5-column ID/Control/Mechanism/Implementation/Source table
  with click-to-expand rows and a colored mechanism pill in the ID column), App UI
  (customer registration form + fleet board grouped by CustomerStatus showing
  profile/welcome/chat-count/eval scores, and a fleet-health panel showing stale
  agents). Browser title exactly:
  <title>Akka Sample: Per-Customer Success CRM Agents</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is
  set, default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options via
  the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider block
        below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf;
        /akka:build forwards the value from the Claude session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory; passed
        to the JVM via the MCP tool's environment parameter; gone when the session
        ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material. Akka records
  only the REFERENCE (env-var name, file path, secrets URI); the value lives in the
  user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error message
  must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing the
  ModelProvider interface with a per-agent dispatch on the agent class name (or the
  Task<R> id). Each agent's branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json, picks one entry
  pseudo-randomly per call, and deserialises it into the agent's typed return
  shape. A MockModelProvider.seedFor(id) helper makes the selection deterministic
  per customerId so the same input produces the same output across restarts.
- Per-agent mock-response shapes for THIS blueprint:
    fleet-coordinator.json — 4-6 entries. REGISTER_CUSTOMER variants are
      FleetEntry with a customerId, name, tier, status SPAWNING, and registeredAt.
      SUMMARIZE_FLEET variants are lists of 1-3 FleetHealthSummary items.
    background-researcher.json — 4-6 CustomerProfile entries, each with a
      customerId, tier, a one-paragraph summary, 3-5 keyFacts, and 2-3
      interactionHints. Include 1 entry whose customerId is mismatched to exercise
      a workflow routing check.
    chat-responder.json — 6-8 ChatReply entries spanning different customer tiers.
      Most are substantive replies. Include 1 entry with an unusually short reply
      (under 20 chars) so FleetEvaluator can flag it. Include 1 entry whose
      sendWelcomeEmail call targets a blocked domain to exercise the G1 before-tool-
      call guardrail refusal.

Constraints — see explainability/specs/eval-matrix/blueprint-program/
AKKA-EXEMPLAR-LESSONS.md for the full list. Notably:
- Run command is "/akka:build" (Claude Code), never "mvn akka:run" (Lesson 9).
- Optional<T> for every nullable field on a View row record (Lesson 6).
- WorkflowSettings is nested inside Workflow — no import needed (Lesson 5).
- emptyState() never calls commandContext() (Lesson 3).
- AutonomousAgent never silently downgraded to Agent (Lesson 1); each
  AutonomousAgent has its companion FleetTasks Task<R> constants (Lesson 7).
- Views have no WHERE filter on the enum status column for fleet status; filter
  client-side (Lesson 2). ChatHistoryView may filter by customerId.
- Workflow steps that call agents set an explicit stepTimeout (Lesson 4).
- Model names verified current before locking application.conf (Lesson 8).
- Explicit http-port in application.conf — 9512 (Lessons 10, 13).
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
