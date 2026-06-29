# SPEC — games-sales-assistant

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** VideoGamesSalesAssistant.
**One-line pitch:** A shopper asks a natural-language question about a video games catalog; one AI agent answers, recommends matching titles, and looks up order history — every response passes through a guardrail that rejects off-catalog references and filters unsafe content before reaching the customer.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the sales-marketing domain. One `SalesAssistantAgent` (AutonomousAgent) handles every customer turn; the surrounding components manage session lifecycle, catalog state, and the read model.  One governance mechanism is wired around the agent:

- A **before-agent-response guardrail** validates every agent turn before the response reaches the caller: all game titles cited must exist in the in-process catalog index, the response must not contain promotional claims outside a defined allowed-claims list, and the response must not include content from a forbidden-topic list (pricing outside the catalog, warranty commitments, personal data of other customers). A rejected turn triggers a retry inside the same task.

The blueprint shows that a recommendation agent serving customer-facing traffic carries material risk even without PII or regulated decisions — an off-catalog title in a recommendation is a broken customer promise, and a forbidden-topic leak is a compliance event. A single guardrail catches both classes of error before they become customer-visible.

## 3. User-facing flows

The user opens the App UI tab.

1. The shopper types a natural-language question into the **Ask** textarea (e.g., "What action RPGs do you have under $40 on PC?") and clicks **Ask**.
2. The UI POSTs to `/api/sessions/{sessionId}/turns` and receives a `turnId`. A session is created automatically on first turn.
3. The conversation card appears in the live list in `TURN_PENDING` state. Within 5–20 s, the agent returns a response. The card transitions to `TURN_ANSWERED`.
4. The agent response renders in the right pane: a plain-language answer paragraph, an optional `recommendations` block (up to 5 `GameTitle` cards with cover art placeholder, title, platform, price, and a one-sentence pitch), and an optional `orders` block if the shopper asked about past purchases.
5. The shopper can type another question in the same session (the agent retains conversation context) or click **New session** to start fresh.
6. A status chip on each session card shows the session lifecycle state.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — create session, post turn, list sessions, get session, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: created → active → closed. Holds conversation history, catalog lookups, order queries. | `SessionEndpoint`, `SessionWorkflow` | `SessionView` |
| `CatalogConsumer` | `Consumer` | Subscribes to `CatalogUpdated` events emitted by the seeded loader; hydrates the in-process `CatalogIndex`. | seeded loader | `CatalogIndex` (in-process) |
| `SessionWorkflow` | `Workflow` | One workflow per session. Steps: `initStep` → `assistStep` (one per turn) → `closeStep`. | started by `SessionEndpoint` | `SalesAssistantAgent`, `SessionEntity` |
| `SalesAssistantAgent` | `AutonomousAgent` | The one decision-making LLM. Receives shopper question plus session context in the task; returns `AssistantResponse`. | invoked by `SessionWorkflow` | returns response |
| `ResponseGuardrail` | supporting class | Before-agent-response hook on `SalesAssistantAgent`. Validates every turn: in-catalog titles only, no forbidden topics, no off-limits promotional claims. | `SalesAssistantAgent` | returns accept or reject |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record GameTitle(
    String titleId,
    String name,
    String platform,
    String genre,
    double priceUsd,
    String shortPitch,
    boolean inStock
) {}

record OrderLine(
    String orderId,
    String titleId,
    String titleName,
    double paidUsd,
    Instant purchasedAt
) {}

record ShopperContext(
    String sessionId,
    String shopperId,
    List<OrderLine> recentOrders,   // last 5, populated by SessionEntity
    List<String> previousTurnIds    // for conversation threading
) {}

record TurnRequest(
    String turnId,
    String question,
    ShopperContext context,
    Instant askedAt
) {}

record Recommendation(
    String titleId,
    String name,
    String platform,
    double priceUsd,
    String pitch                     // one sentence, agent-authored
) {}

record AssistantResponse(
    String answer,
    List<Recommendation> recommendations,   // 0–5 entries
    List<OrderLine> orders,                 // populated only if question was about past orders
    ResponseStatus status,
    Instant answeredAt
) {}
enum ResponseStatus { ANSWERED, GUARDRAIL_REJECTED, AGENT_FAILED }

record Turn(
    String turnId,
    String sessionId,
    TurnRequest request,
    Optional<AssistantResponse> response,
    TurnStatus status,
    Instant createdAt,
    Optional<Instant> answeredAt
) {}
enum TurnStatus { PENDING, ANSWERED, GUARDRAIL_REJECTED, FAILED }

record Session(
    String sessionId,
    String shopperId,
    List<Turn> turns,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}
enum SessionStatus { CREATED, ACTIVE, CLOSED, FAILED }
```

Events on `SessionEntity`: `SessionCreated`, `TurnStarted`, `TurnAnswered`, `TurnRejected`, `TurnFailed`, `SessionClosed`, `SessionFailed`.

Every nullable lifecycle field on `Turn` and `Session` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ shopperId }` → `{ sessionId }`.
- `POST /api/sessions/{sessionId}/turns` — body `{ question }` → `{ turnId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session with all turns.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Video Games Sales Assistant</title>`.

The App UI tab is a two-column layout: a left rail with the live list of sessions (status pill + shopper id + turn count + age) and a right pane with the selected session's conversation view — each turn shows the shopper question, the agent's answer paragraph, the recommendation cards (if any), and the orders block (if any). A status chip per turn indicates `PENDING`, `ANSWERED`, `GUARDRAIL_REJECTED`, or `FAILED`.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-agent-response guardrail**: runs on every turn of `SalesAssistantAgent`. Asserts (1) every `Recommendation.titleId` in the response exists in the in-process `CatalogIndex`, (2) the `answer` text contains no strings from the `FORBIDDEN_TOPICS` list (warranty commitments, competitor pricing, other-customer PII), and (3) any promotional claim in the `answer` is drawn only from the allowed `shortPitch` fields in the catalog. On failure, returns a structured `invalid-response` error to the agent loop naming which check failed, so the task retries within its iteration budget.

## 9. Agent prompts

- `SalesAssistantAgent` → `prompts/sales-assistant.md`. The single decision-making LLM. System prompt instructs it to read the shopper's question, look up matching catalog entries, and return one `AssistantResponse` with a plain-language answer and up to 5 recommendations.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Shopper asks "What action RPGs do you have under $40 on PC?"; within 20 s the response appears with up to 5 matching `GameTitle` recommendations and a plain-language answer paragraph.
2. **J2** — Agent's first turn cites a game title not in the catalog — `ResponseGuardrail` rejects it; the second iteration returns only in-catalog titles; the session card never shows the rejected response.
3. **J3** — Shopper asks about past orders; the agent reads `context.recentOrders` from `ShopperContext` and returns a correct order list.
4. **J4** — Agent attempts to include a warranty commitment phrase in the answer — guardrail rejects on the forbidden-topics check; the card shows `GUARDRAIL_REJECTED` status chip.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named games-sales-assistant demonstrating the single-agent × sales-marketing cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-sales-marketing-games-sales-assistant. Java package
io.akka.samples.videogamessalesassistant. Akka 3.6.0. HTTP port 9306.

Components to wire (exactly):

- 1 AutonomousAgent SalesAssistantAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/sales-assistant.md>) and
  .capability(TaskAcceptance.of(SalesTasks.ASSIST_SHOPPER).maxIterationsPerTask(3)). The task
  receives the shopper question and session context in its instruction text and the serialised
  ShopperContext (JSON) as a task attachment named "context.json". Output:
  AssistantResponse{answer: String, recommendations: List<Recommendation>, orders:
  List<OrderLine>, status: ResponseStatus, answeredAt: Instant}. The agent is configured with a
  before-agent-response guardrail (see G1 in eval-matrix.yaml) registered via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop retries the response
  within its 3-iteration budget.

- 1 Workflow SessionWorkflow per sessionId with three steps:
  * initStep — calls SessionEntity.createSession; records SessionCreated event.
    WorkflowSettings.stepTimeout 5s.
  * assistStep — receives a TurnRequest; emits TurnStarted, then calls
    componentClient.forAutonomousAgent(SalesAssistantAgent.class, "assistant-" + sessionId)
    .runSingleTask(TaskDef.instructions(formatQuestion(turn)).attachment("context.json",
    serializeContext(context).getBytes())) — returns a taskId, then forTask(taskId)
    .result(SalesTasks.ASSIST_SHOPPER) to fetch the AssistantResponse. On success calls
    SessionEntity.recordTurnAnswered(turnId, response). On guardrail failure records
    TurnRejected. WorkflowSettings.stepTimeout 45s with defaultStepRecovery
    maxRetries(2).failoverTo(SessionWorkflow::error).
  * closeStep — emits SessionClosed. WorkflowSettings.stepTimeout 5s.
    error step transitions entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  shopperId: String, turns: List<Turn>, status: SessionStatus, createdAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus enum: CREATED, ACTIVE, CLOSED, FAILED. Events:
  SessionCreated{shopperId}, TurnStarted{request}, TurnAnswered{turnId, response},
  TurnRejected{turnId, reason}, TurnFailed{turnId, reason}, SessionClosed{},
  SessionFailed{reason}. Commands: createSession, startTurn, recordTurnAnswered,
  recordTurnRejected, recordTurnFailed, closeSession, fail, getSession. emptyState() returns
  Session.initial("") with empty turns list and status CREATED, no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 Consumer CatalogConsumer subscribed to CatalogUpdated events; on each event, updates the
  singleton CatalogIndex (an in-process ConcurrentHashMap<String, GameTitle>) with the
  updated catalog entries. The CatalogIndex is injected into ResponseGuardrail for
  title-existence checks.

- 1 View SessionView with row type SessionRow (mirrors Session with turn count, last-turn
  status, and last-turn answeredAt — without full turn bodies to keep the view lean).
  Table updater consumes SessionEntity events. ONE query getAllSessions:
  SELECT * AS sessions FROM session_view. No WHERE status filter — Akka cannot auto-index
  enum columns (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {shopperId}; mints sessionId; starts
    SessionWorkflow.initStep; returns {sessionId}), POST /sessions/{sessionId}/turns (body
    {question}; mints turnId; triggers assistStep; returns {turnId}), GET /sessions (list from
    getAllSessions, newest-first), GET /sessions/{id} (one session with turns), GET
    /sessions/sse (Server-Sent Events from the view stream-updates), and three /api/metadata/*
    endpoints serving the YAML/MD files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SalesTasks.java declaring one Task<R> constant: ASSIST_SHOPPER = Task.name("Assist shopper")
  .description("Answer the shopper's question about the video games catalog and return an
  AssistantResponse with recommendations and optional order history")
  .resultConformsTo(AssistantResponse.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records GameTitle, OrderLine, ShopperContext, TurnRequest, Recommendation,
  AssistantResponse, ResponseStatus, Turn, TurnStatus, Session, SessionStatus.

- ResponseGuardrail.java implementing the before-agent-response hook. Injects CatalogIndex.
  Reads the candidate AssistantResponse from the LLM response, runs three checks: (1) every
  Recommendation.titleId exists in CatalogIndex, (2) answer text contains no string from
  FORBIDDEN_TOPICS set (compile-time constant list), (3) no promotional claim appears that
  is absent from any catalog shortPitch. Returns Guardrail.reject(<structured-error>) naming
  the failing check on any failure; passing responses flow through.

- CatalogIndex.java — a singleton wrapper around ConcurrentHashMap<String, GameTitle> with
  thread-safe upsert and title-existence check methods. Injected via Akka's component
  injection. Pre-populated at startup from src/main/resources/sample-events/catalog.jsonl
  inside CatalogConsumer's startup hook.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9306 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The SalesAssistantAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/catalog.jsonl with 20 seeded GameTitle entries across
  genres (action-RPG, sports, strategy, shooter, puzzle) and platforms (PC, PlayStation,
  Xbox, Switch). Price range $9.99–$69.99. Each title has a realistic shortPitch. Two titles
  marked inStock=false for out-of-stock filtering.

- src/main/resources/sample-events/orders.jsonl with 10 seeded OrderLine entries spread
  across 3 fictional shopperIds (so order-history journeys have data to return).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching Section 8 of this SPEC.
  Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root pre-filled for a retail sales-marketing deployment.

- prompts/sales-assistant.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Video Games Sales Assistant",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of sessions; right = selected-session conversation view with shopper question,
  agent answer paragraph, recommendation cards, and order history block).
  Browser title exactly: <title>Akka Sample: Video Games Sales Assistant</title>. No subtitle
  on the Overview tab.

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

- Generate MockModelProvider.java implementing the ModelProvider interface with per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId + turnId)),
  and deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    assist-shopper.json — 10 AssistantResponse entries covering a range of question types
    (genre filter, price filter, platform filter, order history, combined). Each entry has a
    plain-language answer paragraph, a recommendations list with 1–5 Recommendation entries
    drawn from catalog.jsonl titles, and an orders list (empty unless the question type is
    order-history). Plus 2 deliberately MALFORMED entries (one with a Recommendation.titleId
    that is not in catalog.jsonl; one whose answer text contains a FORBIDDEN_TOPICS string)
    — the guardrail blocks both, exercising the retry path. The mock selects a malformed
    entry on the FIRST iteration of every 3rd session (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(sessionId, turnId) helper makes per-turn selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SalesAssistantAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SalesTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (initStep 5s,
  assistStep 45s, closeStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on Turn and Session is Optional<T>. The view
  table updater wraps values with Optional.of(...); callers use .orElse(...) or .isPresent().
- Lesson 7: SalesTasks.java with ASSIST_SHOPPER = Task.name(...).description(...)
  .resultConformsTo(AssistantResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9306 declared explicitly in application.conf's
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
- The single-agent invariant: there is exactly ONE AutonomousAgent (SalesAssistantAgent).
  No second LLM call is made anywhere in the system.
- The shopper context is passed as a Task ATTACHMENT (context.json), never inlined into
  the agent's instruction text — keeping large order-history payloads out of the prompt.
- The before-agent-response guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent returns.
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
