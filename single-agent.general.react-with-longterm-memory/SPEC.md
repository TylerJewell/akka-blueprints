# SPEC — mem0-react-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Mem0ReactAgent.
**One-line pitch:** A user sends a question; one ReAct agent reasons over a tool set and, when it learns a fact about the user, persists it to an in-process long-term memory store — so future sessions recall what past sessions discovered.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `Mem0ReactAgent` (AutonomousAgent) carries the entire decision loop — tool selection, reasoning, memory recall, and answer synthesis. The surrounding components handle persistence and governance. Two governance mechanisms sit around the agent:

- A **PII sanitizer** runs inside a Consumer between the agent's raw fact-store request and the actual persistence write — so identifiers are redacted before any fact lands in long-term memory.
- A **human-on-the-loop monitor** subscribes to `FactPersisted` events; when a user's cumulative fact count crosses a configurable threshold it emits a `MemoryDriftSignaled` event that a deployer-operated dashboard can surface for review.

The blueprint shows that a ReAct agent with persistent memory is not ungoverned: facts are cleaned before they are written, and growth is observable without blocking the agent's loop.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a question into the **Message** input and clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/turns` and receives a `turnId`.
2. The in-progress card appears in the live session list. Within ~10–30 s the agent finishes its ReAct loop and the answer appears. Any tool calls the agent made are visible as collapsible steps inside the turn card.
3. If the agent discovered a new fact about the user while answering (e.g., the user mentioned a preference), the agent calls the `store-memory` tool. Within ~1 s the fact transitions through `SANITIZE_PENDING → SANITIZED → PERSISTED`.
4. On the second session (or after clicking **New session**), the user asks a follow-up that requires the stored fact. The agent calls `recall-memories` first, finds the fact, and weaves it into its answer without asking the user again.
5. When a user's fact count reaches the configurable drift threshold (default: 50 facts), a `MemoryDriftSignaled` event surfaces in the **Monitor** panel on the App UI tab.
6. The user can start additional sessions; each session has its own conversation history but shares the same long-term memory store across sessions.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `AgentEndpoint` | `HttpEndpoint` | `/api/sessions/*` and `/api/memory/*` — start session, post turn, list facts, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `MemoryEntity`, `SessionView`, `MemoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: open → active → closed. Holds turn history. | `AgentEndpoint`, `Mem0ReactAgent` | `SessionView` |
| `MemoryEntity` | `EventSourcedEntity` | Per-fact lifecycle: sanitize-pending → sanitized → persisted. One entity per factId. | `FactSanitizer`, `MemoryWriteWorkflow` | `MemoryView` |
| `FactSanitizer` | `Consumer` | Subscribes to `FactStoreRequested` events; redacts PII; calls `MemoryEntity.attachSanitized`. | `MemoryEntity` events | `MemoryEntity` |
| `MemoryWriteWorkflow` | `Workflow` | One workflow per factId. Steps: `awaitSanitizedStep` → `persistStep`. | started by `FactSanitizer` once `FactSanitized` lands | `MemoryEntity` |
| `DriftMonitor` | `Consumer` | Subscribes to `FactPersisted` events; counts facts per userId; emits `MemoryDriftSignaled` when threshold crossed. | `MemoryEntity` events | `MemoryView` |
| `Mem0ReactAgent` | `AutonomousAgent` | The one decision-making LLM. Runs a ReAct loop over tools (`recall-memories`, `store-memory`, `web-search`, `calculate`). Returns `AgentAnswer` per turn. | invoked by `AgentEndpoint` per turn | returns answer, stores facts |
| `SessionView` | `View` | Read model: one row per session turn for the UI. | `SessionEntity` events | `AgentEndpoint` |
| `MemoryView` | `View` | Read model: one row per fact + drift signals. | `MemoryEntity` events | `AgentEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record UserMessage(String sessionId, String userId, String text, Instant sentAt) {}

record ToolCall(String toolName, String input, String output, Instant calledAt) {}

record AgentAnswer(
    String turnId,
    String answer,
    List<ToolCall> toolCalls,
    Instant answeredAt
) {}

record MemoryFact(
    String factId,
    String userId,
    String rawText,
    Instant requestedAt
) {}

record SanitizedFact(
    String cleanText,
    List<String> piiCategoriesFound
) {}

record DriftSignal(
    String userId,
    int factCount,
    int threshold,
    Instant signaledAt
) {}

record Session(
    String sessionId,
    String userId,
    List<UserMessage> turns,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}
enum SessionStatus { OPEN, ACTIVE, CLOSED }

record Memory(
    String factId,
    String userId,
    Optional<MemoryFact> request,
    Optional<SanitizedFact> sanitized,
    MemoryStatus status,
    Instant createdAt,
    Optional<Instant> persistedAt
) {}
enum MemoryStatus { SANITIZE_PENDING, SANITIZED, PERSISTED, FAILED }
```

Events on `SessionEntity`: `SessionOpened`, `TurnStarted`, `TurnAnswered`, `SessionClosed`.
Events on `MemoryEntity`: `FactStoreRequested`, `FactSanitized`, `FactPersisted`, `FactFailed`.
Events on `DriftMonitor` (published downstream): `MemoryDriftSignaled`.

Every nullable lifecycle field on `Memory` and `Session` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ userId }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions newest-first.
- `GET /api/sessions/{id}` — one session with turn history.
- `POST /api/sessions/{id}/turns` — body `{ text }` → `{ turnId }`. Triggers the agent.
- `GET /api/sessions/sse` — Server-Sent Events; one event per session or turn state transition.
- `GET /api/memory/{userId}` — list all persisted facts for a user.
- `GET /api/memory/sse` — Server-Sent Events for fact lifecycle and drift signals.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Mem0 ReAct Agent</title>`.

The App UI tab is a two-column layout: a left column with the active session's conversation thread (user messages + agent answers with collapsible tool-call steps), and a right column with the long-term memory panel (list of persisted facts per user + a drift monitor strip showing `MemoryDriftSignaled` events).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `FactSanitizer` Consumer): before any fact text reaches the persistent store, the Consumer redacts emails, phone numbers, government identifiers, payment-card-like tokens, person names, postal addresses, and account-like tokens. Records which categories were found. The `MemoryWriteWorkflow.persistStep` only persists `sanitized.cleanText`, never `rawText`.
- **H1 — Human-on-the-loop drift monitor** (`hotl`, `deployer-runtime-monitoring`): `DriftMonitor` Consumer maintains a per-user count of persisted facts (read from `MemoryView`). When the count crosses the configurable threshold (default 50, overridable in `application.conf`), it emits a `MemoryDriftSignaled` event. The App UI's memory panel shows these signals as warning banners. No agent call is blocked — the monitor is non-blocking by design.

## 9. Agent prompts

- `Mem0ReactAgent` → `prompts/mem0-react-agent.md`. The single decision-making LLM. System prompt defines the ReAct loop, the four tools (`recall-memories`, `store-memory`, `web-search`, `calculate`), and the rules for when to persist a fact versus when to just use it in the answer.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends a preference in session 1; a fact is stored and sanitized; in session 2 the agent recalls it and uses it without being told again.
2. **J2** — A fact containing a raw email address is stored; the persisted `cleanText` shows `[REDACTED-EMAIL]`, never the original string.
3. **J3** — A user's fact count exceeds the drift threshold; a `MemoryDriftSignaled` event appears in the monitor panel within 1 s of the persisting event.
4. **J4** — The agent answers a multi-step arithmetic question by emitting a `calculate` tool call with the expression; the answer matches the correct result.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mem0-react-agent demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-react-with-longterm-memory. Java package
io.akka.samples.mem0reactagent. Akka 3.6.0. HTTP port 9621.

Components to wire (exactly):

- 1 AutonomousAgent Mem0ReactAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/mem0-react-agent.md>) and
  .capability(TaskAcceptance.of(ANSWER_TURN).maxIterationsPerTask(8)). The task receives
  the user message text and the userId in its instructions, plus a tool set of four tools:
  recall-memories, store-memory, web-search, calculate. Each tool is implemented as a
  named function the agent can invoke in its ReAct loop. The agent is configured WITHOUT
  a guardrail — governance here is at the memory write path, not the answer path.

- 1 Workflow MemoryWriteWorkflow per factId with two steps:
  * awaitSanitizedStep — polls MemoryEntity.getMemory every 1s; on
    memory.sanitized().isPresent() advances to persistStep. WorkflowSettings.stepTimeout
    15s (sanitizer is in-process and fast).
  * persistStep — calls MemoryEntity.persist(factId). Emits FactPersisted.
    WorkflowSettings.stepTimeout 10s with defaultStepRecovery maxRetries(2)
    .failoverTo(MemoryWriteWorkflow::error). error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  userId: String, turns: List<UserMessage>, status: SessionStatus, createdAt: Instant,
  closedAt: Optional<Instant>}. SessionStatus: OPEN, ACTIVE, CLOSED. Events:
  SessionOpened{userId}, TurnStarted{turnId, text}, TurnAnswered{answer: AgentAnswer},
  SessionClosed{}. Commands: open, startTurn, recordAnswer, close, getSession.
  emptyState() returns Session.initial("") with empty turns list (Lesson 3).

- 1 EventSourcedEntity MemoryEntity (one per factId). State Memory{factId: String,
  userId: String, request: Optional<MemoryFact>, sanitized: Optional<SanitizedFact>,
  status: MemoryStatus, createdAt: Instant, persistedAt: Optional<Instant>}. MemoryStatus:
  SANITIZE_PENDING, SANITIZED, PERSISTED, FAILED. Events: FactStoreRequested{request},
  FactSanitized{sanitized}, FactPersisted{}, FactFailed{reason}. Commands: requestStore,
  attachSanitized, persist, fail, getMemory. emptyState() returns Memory.initial("") with
  all Optional fields as Optional.empty() (Lesson 3).

- 1 Consumer FactSanitizer subscribed to MemoryEntity events; on FactStoreRequested runs
  a regex+heuristic redaction pipeline (emails, phone numbers, SSN-like, payment-card-like,
  postal addresses, person-name heuristic, account-id-like tokens) over rawText, computes
  the list of categories found, builds SanitizedFact, then calls
  MemoryEntity.attachSanitized(sanitized). After attachSanitized lands, the same Consumer
  starts a MemoryWriteWorkflow with id = "mem-" + factId.

- 1 Consumer DriftMonitor subscribed to MemoryEntity events; on FactPersisted queries
  MemoryView.countByUserId(userId) and compares against the configured threshold
  (akka.javasdk.agent.memory.drift-threshold, default 50). If count >= threshold, writes
  a MemoryDriftSignaled{userId, factCount, threshold, signaledAt} event into MemoryView
  (via a dedicated MemoryEntity command or a separate lightweight DriftSignalEntity —
  implementer's choice, but the event must be queryable from MemoryView).

- 1 View SessionView with row type SessionRow (mirrors Session). ONE query
  getAllSessions: SELECT * AS sessions FROM session_view. No WHERE status filter (Lesson 2).

- 1 View MemoryView with row type MemoryRow (mirrors Memory minus rawText). Separate
  query getDriftSignals: SELECT * AS signals FROM drift_signal_view. ONE query
  getFactsByUser: SELECT * AS facts FROM memory_view WHERE userId = :userId.

- 2 HttpEndpoints:
  * AgentEndpoint at /api with POST /sessions (body {userId}; mints sessionId; calls
    SessionEntity.open; returns {sessionId}), GET /sessions (list from getAllSessions,
    sorted newest-first), GET /sessions/{id} (one row), POST /sessions/{id}/turns (body
    {text}; mints turnId; calls SessionEntity.startTurn; triggers Mem0ReactAgent
    runSingleTask; on completion calls SessionEntity.recordAnswer; returns {turnId}),
    GET /sessions/sse (Server-Sent Events forwarded from session view's stream-updates),
    GET /memory/{userId} (list facts from getFactsByUser), GET /memory/sse (SSE from
    memory view stream-updates), and three /api/metadata/* endpoints serving the YAML/MD
    files from src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* ->
    static-resources/*.

Companion files:

- AgentTasks.java declaring one Task<R> constant: ANSWER_TURN = Task.name("Answer turn")
  .description("Reason over the user message using available tools; recall and store
  memory facts as needed; return a complete answer").resultConformsTo(AgentAnswer.class).
  DO NOT skip this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records UserMessage, ToolCall, AgentAnswer, MemoryFact, SanitizedFact,
  DriftSignal, Session, Memory, SessionStatus, MemoryStatus.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9621,
  akka.javasdk.agent.memory.drift-threshold = 50, and the three model-provider blocks
  (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini gemini-2.5-flash) reading
  the canonical env vars ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-facts.jsonl with 6 seeded user-fact strings
  covering preferences, location, interests, and one containing PII (email + phone) to
  exercise the sanitizer.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (S1, H1) matching the mechanisms
  in Section 8 of this SPEC.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = false (PII goes through the LLM when the user
  first mentions it — the sanitizer only cleans facts before they are persisted),
  decisions.authority_level = autonomous (the agent answers without human approval per
  turn), oversight.human_on_loop = true (the drift monitor surfaces signals for deployer
  review), failure.failure_modes including "stale-memory-recall",
  "pii-persisted-before-sanitize", "memory-drift-undetected", "hallucinated-tool-call";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/mem0-react-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Mem0 ReAct Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  conversation thread with user messages and agent answers; right = memory panel with
  persisted facts list and drift monitor strip).
  Browser title exactly: <title>Akka Sample: Mem0 ReAct Agent</title>. No subtitle on the
  Overview tab.

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
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. For ANSWER_TURN: reads
  src/main/resources/mock-responses/answer-turn.json, picks one entry pseudo-randomly per
  call (seedFor(sessionId + turnId)), and deserialises into AgentAnswer. Include 6
  AgentAnswer entries: 4 that include at least one store-memory tool call (so the memory
  path exercises), 1 that includes a calculate tool call, and 1 that includes a
  recall-memories tool call returning a fact from the seed set. MockModelProvider.seedFor
  makes selection deterministic per session+turn.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. Mem0ReactAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion AgentTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (awaitSanitizedStep 15s,
  persistStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on Memory and Session uses Optional<T>; view
  updater wraps with Optional.of(...).
- Lesson 7: AgentTasks.java with ANSWER_TURN is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build".
- Lesson 10: port 9621 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  AND the mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (Mem0ReactAgent). The
  DriftMonitor is a rule-based Consumer — no LLM call.
- Facts are passed to MemoryWriteWorkflow after the agent's store-memory tool call; the
  agent NEVER writes directly to MemoryEntity — it goes through the tool call handler
  which calls MemoryEntity.requestStore, triggering the sanitize path.
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
