# SPEC — long-term-memory-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MemoryAgent.
**One-line pitch:** A user sends a message to an agent that recalls facts from past sessions, generates a contextually grounded reply, and writes new memory entries that persist — sanitized for PII — across every future session.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `MemoryAgent` (AutonomousAgent) handles every turn: it receives the current message plus a set of recalled memory entries, generates a reply, and produces candidate memory entries to store. The surrounding components manage recall, sanitization, and persistence.

One governance mechanism is wired around the agent:

- A **PII sanitizer** runs inside a Consumer between the raw extracted memory entries and the persistence write — so names, contact details, and account identifiers gathered during a conversation are never stored verbatim and never injected into future model calls in raw form.

The blueprint shows that long-term memory does not require a second model call to govern. The sanitizer is a deterministic pipeline; the single agent remains the only component that talks to a model.

## 3. User-facing flows

The user opens the App UI tab.

1. The user selects or creates a **session** from a left-rail list. Each session belongs to a `userId` (typed into a text field). New sessions start in `ACTIVE` state.
2. The user types a message in the **Conversation** panel and clicks **Send**. The UI POSTs to `/api/sessions/{sessionId}/turns` and receives a `turnId`.
3. Within ~1 s, the agent recalls the user's top-K memories and formulates a reply. The reply appears in the conversation thread.
4. After the reply, the agent produces candidate memory entries (facts worth remembering). These are emitted as `MemoryExtracted` events, which the `MemorySanitizer` picks up, scrubs, and stores as `MemoryEntry` records on `MemoryEntity`.
5. The memory panel on the right shows the current user's stored memories — newest-first, with a small PII-scrubbed label where redaction occurred.
6. When the session reaches its turn limit it transitions to `CLOSED`. The closed session remains readable.
7. The user can start a new session; the agent's recall step immediately surfaces memories from previous sessions.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `ConversationEndpoint` | `HttpEndpoint` | `/api/sessions/*` and `/api/memories/*` — create, list, get, send turn, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `MemoryEntity`, `SessionView`, `MemoryView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: active → closing → closed. Holds conversation turns. Source of truth for sessions. | `ConversationEndpoint`, `MemoryExtractionWorkflow` | `SessionView` |
| `MemoryEntity` | `EventSourcedEntity` | Per-user persistent memory store. Holds sanitized `MemoryEntry` records. | `MemorySanitizer` | `MemoryView` |
| `MemorySanitizer` | `Consumer` | Subscribes to `MemoryExtracted` events; scrubs PII from raw memory text; calls `MemoryEntity.storeMemory`. | `SessionEntity` events | `MemoryEntity` |
| `MemoryExtractionWorkflow` | `Workflow` | One workflow per turn. Steps: `recallStep` → `respondStep` → `extractStep`. | started by `ConversationEndpoint` on turn creation | `MemoryAgent`, `SessionEntity`, `MemoryView` |
| `MemoryAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user message and recalled memories; returns `AgentReply` containing the response text and candidate memory entries. | invoked by `MemoryExtractionWorkflow` | returns `AgentReply` |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `ConversationEndpoint` |
| `MemoryView` | `View` | Read model: one row per memory entry per user. | `MemoryEntity` events | `ConversationEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record ConversationTurn(
    String turnId,
    String message,
    String reply,
    Instant sentAt,
    Instant repliedAt,
    TurnStatus status
) {}
enum TurnStatus { PENDING, REPLIED, FAILED }

record RawMemoryEntry(
    String entryId,
    String rawText,          // unredacted — never stored to MemoryEntity
    MemoryKind kind,
    float relevanceScore
) {}
enum MemoryKind { FACT, PREFERENCE, RELATIONSHIP, CONTEXT }

record MemoryEntry(
    String entryId,
    String userId,
    String sanitizedText,    // PII-scrubbed form; stored and recalled
    List<String> piiCategoriesRedacted,
    MemoryKind kind,
    float relevanceScore,
    Instant recordedAt,
    Optional<Instant> expiresAt
) {}

record AgentReply(
    String replyText,
    List<RawMemoryEntry> candidateMemories,
    Instant generatedAt
) {}

record Session(
    String sessionId,
    String userId,
    List<ConversationTurn> turns,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> closedAt
) {}

enum SessionStatus { ACTIVE, CLOSING, CLOSED }

record UserMemoryStore(
    String userId,
    List<MemoryEntry> memories,
    Instant lastUpdatedAt
) {}
```

Events on `SessionEntity`: `SessionCreated`, `TurnAdded`, `ReplyRecorded`, `MemoryExtracted`, `SessionClosed`, `TurnFailed`.
Events on `MemoryEntity`: `MemoryStored`, `MemoryExpired`.

Every nullable lifecycle field on `Session` and `MemoryEntry` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ userId, sessionLabel? }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions for a userId, newest-first. Query param `?userId=`.
- `GET /api/sessions/{id}` — one session with full turn history.
- `GET /api/sessions/{id}/turns` — SSE stream of turn updates for one session.
- `POST /api/sessions/{id}/turns` — body `{ message, userId }` → `{ turnId }`.
- `GET /api/memories` — list stored memories for a user. Query param `?userId=`.
- `GET /api/sessions/sse` — Server-Sent Events; one event per session state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Agent with Long-Term Memory</title>`.

The App UI tab is a two-column layout: a left rail with session list + new-session form, and a right pane with the active session's conversation thread, the agent's reply, and a memory panel showing stored entries for that user.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **S1 — PII sanitizer** (`pii`, applied inside `MemorySanitizer` Consumer): redacts emails, phone numbers, government identifiers, person names, postal addresses, and account-like tokens from raw memory text before any write to `MemoryEntity`. The `RawMemoryEntry` list produced by the agent is never persisted; only the `MemoryEntry` with `sanitizedText` is stored. Records which categories were redacted.

## 9. Agent prompts

- `MemoryAgent` → `prompts/memory-agent.md`. The single decision-making LLM. System prompt instructs it to read the user's recalled memories, respond to the current message in light of that context, and extract a list of candidate memory entries from the conversation for future recall.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User sends an opening message; agent replies and emits memory candidates; sanitized entries appear in the memory panel within 5 s.
2. **J2** — User starts a second session with the same userId; the agent's recall step surfaces at least one fact from the first session and references it in its reply.
3. **J3** — A message containing `alice.jones@example.com` and `SSN 987-65-4321` produces memory entries where those strings are replaced by `[REDACTED-EMAIL]` and `[REDACTED-SSN]`; the raw strings never appear in `MemoryEntity` events.
4. **J4** — After a session accumulates its maximum turns it transitions to `CLOSED`; a subsequent POST to that session returns `409 Conflict`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named long-term-memory-agent demonstrating the single-agent × general cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-long-term-memory-agent. Java package
io.akka.samples.agentwithlongtermmemory. Akka 3.6.0. HTTP port 9889.

Components to wire (exactly):

- 1 AutonomousAgent MemoryAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/memory-agent.md>) and
  .capability(TaskAcceptance.of(CONVERSE_WITH_MEMORY).maxIterationsPerTask(2)).
  Inputs: the user's current message in the task instructions text; the recalled
  MemoryEntry list serialized as a JSON attachment named "memories.json" (a Task
  ATTACHMENT, NOT inline prompt text — Akka's TaskDef.attachment(name, contentBytes)
  is the canonical call). Output: AgentReply{replyText: String, candidateMemories:
  List<RawMemoryEntry>, generatedAt: Instant}. No guardrail on this agent (the
  memory extraction path is additive, not decision-gating — a bad parse simply
  produces no new memories, not a bad outcome).

- 1 Workflow MemoryExtractionWorkflow per turnId with three steps:
  * recallStep — queries MemoryView for the userId's memories, selects top-K by
    relevanceScore (K defaults to 10), wraps them as the "memories.json" attachment.
    WorkflowSettings.stepTimeout 5s (view query is in-process and fast).
  * respondStep — emits TurnAdded to SessionEntity, then calls
    componentClient.forAutonomousAgent(MemoryAgent.class, "agent-" + userId)
    .runSingleTask(TaskDef.instructions(userMessage)
      .attachment("memories.json", recalledMemoriesJson.getBytes()))
    — returns a taskId, then forTask(taskId).result(CONVERSE_WITH_MEMORY) to fetch
    the AgentReply. On success calls SessionEntity.recordReply(turnId, replyText)
    and SessionEntity.emitMemoryExtracted(turnId, agentReply.candidateMemories).
    WorkflowSettings.stepTimeout 60s with defaultStepRecovery maxRetries(2)
    .failoverTo(MemoryExtractionWorkflow::error).
  * extractStep — iterates agentReply.candidateMemories; for each entry publishes a
    MemoryExtracted event on the SessionEntity carrying the RawMemoryEntry. The
    MemorySanitizer Consumer picks these up asynchronously. extractStep itself
    completes synchronously after emitting all events. WorkflowSettings.stepTimeout 5s.
    error step transitions the turn to FAILED via SessionEntity.failTurn(turnId, reason).

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 2 EventSourcedEntities:
  * SessionEntity (one per sessionId). State Session{sessionId: String, userId: String,
    turns: List<ConversationTurn>, status: SessionStatus, createdAt: Instant,
    closedAt: Optional<Instant>}. SessionStatus enum: ACTIVE, CLOSING, CLOSED. Events:
    SessionCreated{userId}, TurnAdded{turnId, message, sentAt}, ReplyRecorded{turnId,
    replyText, repliedAt}, MemoryExtracted{turnId, entries: List<RawMemoryEntry>},
    SessionClosed{closedAt}, TurnFailed{turnId, reason}. Commands: create, addTurn,
    recordReply, emitMemoryExtracted, close, failTurn, getSession. Max turns per
    session = 50; addTurn rejects if status != ACTIVE or turns.size() >= 50.
    emptyState() returns Session.initial("","") with empty turns list, status=ACTIVE,
    closedAt=Optional.empty(). Never references commandContext() in emptyState()
    (Lesson 3).
  * MemoryEntity (one per userId). State UserMemoryStore{userId: String,
    memories: List<MemoryEntry>, lastUpdatedAt: Instant}. Events: MemoryStored{entry},
    MemoryExpired{entryId}. Commands: storeMemory, expireMemory, getMemories.
    emptyState() returns UserMemoryStore.initial("") with empty memories list (Lesson 3).

- 1 Consumer MemorySanitizer subscribed to SessionEntity events; on MemoryExtracted
  iterates each RawMemoryEntry, runs a regex+heuristic redaction pipeline (emails, phone
  numbers, SSN-like, postal addresses, person-name heuristic, account-id-like tokens) over
  rawText, records piiCategoriesRedacted, builds MemoryEntry with sanitizedText, then calls
  MemoryEntity.storeMemory(entry) for each. The raw RawMemoryEntry is never written to
  MemoryEntity.

- 2 Views:
  * SessionView with row type SessionRow (mirrors Session, includes turns list). ONE query
    getSessionsByUser: SELECT * AS sessions FROM session_view WHERE userId = :userId. One
    additional query getSession: SELECT * AS session FROM session_view WHERE sessionId = :sessionId.
  * MemoryView with row type MemoryRow (mirrors MemoryEntry). ONE query getMemoriesByUser:
    SELECT * AS memories FROM memory_view WHERE userId = :userId ORDER BY recordedAt DESC.

- 2 HttpEndpoints:
  * ConversationEndpoint at /api with:
    POST /sessions (body {userId, sessionLabel?}; mints sessionId; calls SessionEntity.create;
      returns {sessionId}),
    GET /sessions?userId= (list from getSessionsByUser, sorted newest-first),
    GET /sessions/{id} (one session from getSession),
    GET /sessions/{id}/turns (SSE stream for one session's turn updates),
    POST /sessions/{id}/turns (body {message, userId}; mints turnId; starts
      MemoryExtractionWorkflow; returns {turnId}),
    GET /memories?userId= (list from getMemoriesByUser),
    GET /sessions/sse (Server-Sent Events forwarded from SessionView stream-updates),
    and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- MemoryTasks.java declaring one Task<R> constant: CONVERSE_WITH_MEMORY = Task
  .name("Converse with memory")
  .description("Reply to the user's message using recalled memories, then extract new memory candidates")
  .resultConformsTo(AgentReply.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records ConversationTurn, TurnStatus, RawMemoryEntry, MemoryKind, MemoryEntry,
  AgentReply, Session, SessionStatus, UserMemoryStore.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9889 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The MemoryAgent.definition() binds
  the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/seed-conversations.jsonl with 3 seeded conversation
  sequences: a technical support exchange (5 turns), a project planning exchange (4 turns),
  and a personal assistant exchange (3 turns). Each exchange contains 2–3 plausible PII
  strings so S1 has work to do.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (S1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = true,
  pii_handled_by_sanitizer_before_llm = true (sanitizer runs between agent output and
  memory storage, not before the agent call), decisions.authority_level = conversational
  (the agent generates dialogue, not enforced decisions), oversight.human_in_loop = false
  (the agent responds autonomously; a human reviews stored memories at their discretion),
  failure.failure_modes including "pii-in-stored-memory", "stale-memory-recall",
  "irrelevant-memory-injection", "session-context-loss"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/memory-agent.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Agent with Long-Term Memory",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms
  section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  session list + new-session form; right = active session's conversation thread + memory
  panel). Browser title exactly: <title>Akka Sample: Agent with Long-Term Memory</title>.
  No subtitle on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        structurally-valid AgentReply outputs. Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in application.conf.
    (c) Point to an existing env file — record the PATH in a project-local .akka-build.yaml.
    (d) Secrets-store URI — recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives only in Claude session memory; passed to
        the JVM via the MCP tool's environment parameter; gone when the session ends.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(userId + turnId)),
  and deserialises into AgentReply.
- Per-task mock-response shapes for THIS blueprint:
    converse-with-memory.json — 6 AgentReply entries. Each has a replyText (1–3
    sentences referencing at least one of the recalled memories) and a candidateMemories
    list (1–3 RawMemoryEntry items with realistic facts, preferences, or context, each
    with a MemoryKind and a relevanceScore between 0.6 and 1.0). Include 1 entry whose
    candidateMemories contain a raw email address and a phone number — exercising the
    sanitizer path.
- A MockModelProvider.seedFor(userId, turnId) helper makes per-turn selection
  deterministic across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. MemoryAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion MemoryTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (respondStep
  60s, recallStep 5s, extractStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field (closedAt on Session, expiresAt on MemoryEntry)
  is Optional<T>. The view table updater wraps values with Optional.of(...); callers use
  .orElse(...) or .isPresent().
- Lesson 7: MemoryTasks.java with CONVERSE_WITH_MEMORY = Task.name(...).description(...)
  .resultConformsTo(AgentReply.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9889 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any
  user-visible string.
- Lesson 23: no forbidden words in narrative.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block (nodeTextColor, stateLabelColor,
  transitionLabelColor #cccccc). Without these, state names render black-on-black and
  arrow labels clip at the top.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. The DOM contains exactly five <section class="tab-panel"> elements — Overview,
  Architecture, Risk Survey, Eval Matrix, App UI — no more.
- The single-agent invariant: there is exactly ONE AutonomousAgent (MemoryAgent). The
  MemorySanitizer is a deterministic pipeline — NOT an LLM — keeping the "one agent"
  promise honest.
- The recalled memories are passed as a Task ATTACHMENT ("memories.json"), never inlined
  into the agent's instructions text. Verify the generated respondStep uses
  TaskDef.attachment(...) and not string interpolation.
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
