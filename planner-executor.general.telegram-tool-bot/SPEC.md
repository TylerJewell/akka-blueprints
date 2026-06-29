# SPEC — telegram-tool-bot

The natural-language brief `/akka:specify` reads to generate this system. Detail wins.

---

## 1. System name + pitch

**System name:** Agentic Telegram AI Bot.
**One-line pitch:** Send a Telegram message; a Router plans the work on a session ledger, dispatches each step to one of four tool-executor agents (web lookup, contact book, calendar, note-saver), records the outcome on a tool ledger, and replies to the user when the goal is reached.

## 2. What this blueprint demonstrates

The **planner-executor** coordination pattern with multiple specialist tool executors behind a conversational front door. The Router owns two ledgers — a **session ledger** (parsed intent, facts known, tool plan, current dispatch) and a **tool ledger** (each tool call's attempt count, verdict, raw result before sanitization, scrubbed result after). Each loop iteration the Router reads both ledgers, picks the next tool executor, and either continues, replans, replies, or fails. On three consecutive failures of the same subtask, or two consecutive replans without forward progress, the Router emits a terminal failure.

The blueprint also demonstrates one governance control wired into that loop:

- a **before-tool-call guardrail** that vets each tool dispatch against an allowlist and a content policy, blocking dispatches to un-registered tool executors or calls whose payload violates the scope policy.

## 3. User-facing flows

The user opens the App UI tab and submits a simulated Telegram message via the form (or waits for the simulator to drip one).

1. The system creates a `Session` record in `PLANNING` and starts a `SessionWorkflow`.
2. The RouterAgent drafts a `SessionLedger { intent, facts, toolPlan, currentDispatch }` and emits `SessionPlanned`.
3. The workflow enters the executor loop. Each iteration:
   - RouterAgent reads both ledgers and proposes a `ToolDispatch { tool, subtask, rationale }`.
   - The **before-tool-call guardrail** vets the dispatch; on rejection the workflow records a `ToolCallBlocked` entry on the tool ledger and asks the Router to revise.
   - The chosen tool executor runs the subtask and returns a typed `ToolResult`.
   - The **secret sanitizer** scrubs the result.
   - The workflow appends a `ToolEntry { tool, subtask, attempt, verdict, scrubbedResult }` to the tool ledger.
4. The RouterAgent decides on each tick: `CONTINUE`, `REPLAN`, `REPLY`, or `FAIL`. After three consecutive failures on the same subtask or two consecutive replans, the Router emits `FAIL`.
5. On `REPLY`, the RouterAgent produces a `BotReply { text, citations }` and emits `SessionCompleted`. The Session moves to `COMPLETED`.
6. The operator can press **Pause new dispatches** in the dashboard at any time. The workflow finishes the in-flight tool call, then ends with `SessionPausedOperator`. The Session moves to `PAUSED`.

A `MessageSimulator` (TimedAction) drips a sample Telegram message every 75 seconds so the App UI is not empty on first load.

## 4. Components

The generated system has exactly the following components. Names matter — `/akka:specify` and `/akka:implement` use them verbatim.

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `RouterAgent` | `AutonomousAgent` | Plans, dispatches, decides on each loop tick. Maintains session ledger; reads tool ledger. Produces `BotReply` on completion. | `SessionWorkflow` | returns typed result to workflow |
| `WebLookupAgent` | `AutonomousAgent` | Answers web-search subtasks from seeded fixtures (`sample-data/web-fixtures.jsonl`). | `SessionWorkflow` | — |
| `ContactBookAgent` | `AutonomousAgent` | Queries contact fixture store (`sample-data/contacts.jsonl`) and returns matching records. | `SessionWorkflow` | — |
| `CalendarQueryAgent` | `AutonomousAgent` | Returns availability and event data from fixture calendars (`sample-data/calendar-fixtures.jsonl`). | `SessionWorkflow` | — |
| `NoteSaverAgent` | `AutonomousAgent` | Persists a short note payload to the fixture notes store (`sample-data/notes-store.jsonl`). Returns a save confirmation. | `SessionWorkflow` | — |
| `SessionWorkflow` | `Workflow` | Drives the plan → dispatch-guarded → execute → sanitize → record → decide loop, plus replan and pause branches. | `SessionEndpoint`, `MessageConsumer` | `SessionEntity` |
| `SessionEntity` | `EventSourcedEntity` | Holds the session lifecycle, session ledger, tool ledger, and final reply. | `SessionWorkflow` | `SessionView` |
| `BotControlEntity` | `EventSourcedEntity` | Holds the operator pause flag. Single instance keyed by literal `"global"`. | `SessionEndpoint` (operator action) | `SessionWorkflow` (polls) |
| `MessageQueue` | `EventSourcedEntity` | Audit log of inbound messages. | `SessionEndpoint`, `MessageSimulator` | `MessageConsumer` |
| `SessionView` | `View` | List-of-sessions read model for the UI. | `SessionEntity` events | `SessionEndpoint` |
| `MessageConsumer` | `Consumer` | Subscribes to `MessageQueue` events; starts a `SessionWorkflow` per message received. | `MessageQueue` events | `SessionWorkflow` |
| `MessageSimulator` | `TimedAction` | Every 75 s, reads a line from `sample-events/message-prompts.jsonl` and enqueues it. | scheduler | `MessageQueue` |
| `StaleSessionMonitor` | `TimedAction` | Every 30 s, marks any session stuck in `EXECUTING` past 4 minutes as `STALE`. The workflow polls this and ends with `SessionFailedTimeout`. | scheduler | `SessionEntity` |
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit message, get, list, SSE, operator pause/resume. | — | `SessionView`, `MessageQueue`, `SessionEntity`, `BotControlEntity` |
| `AppEndpoint` | `HttpEndpoint` | Serves the embedded UI and `/api/metadata/*`. | — | static resources |

## 5. Data model

Authoritative — `/akka:implement` produces these exactly as written. Use `Optional<T>` for any lifecycle field that is null before its transition (Lesson 6 of the exemplar).

### Records

```java
record MessageRequest(String text, String chatId) {}

record SessionLedger(
    String intent,
    List<String> facts,
    List<String> toolPlan,
    Optional<ToolDispatch> currentDispatch
) {}

record ToolDispatch(
    ToolKind tool,
    String subtask,
    String rationale
) {}

record ToolResult(
    ToolKind tool,
    String subtask,
    boolean ok,
    String content,
    Optional<String> errorReason
) {}

record ToolEntry(
    int attempt,
    ToolKind tool,
    String subtask,
    ToolVerdict verdict,
    String scrubbedResult,
    Optional<String> blocker,
    Instant recordedAt
) {}

record ToolLedger(List<ToolEntry> entries) {}

record BotReply(String text, List<String> citations, Instant producedAt) {}

record Session(
    String sessionId,
    String chatId,
    String text,
    SessionStatus status,
    Optional<SessionLedger> ledger,
    Optional<ToolLedger> toolLedger,
    Optional<BotReply> reply,
    Optional<String> failureReason,
    Optional<String> pauseReason,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum ToolKind { WEB, CONTACTS, CALENDAR, NOTES }
enum ToolVerdict { OK, BLOCKED_BY_GUARDRAIL, FAILED, UNSAFE }
enum SessionStatus { PLANNING, EXECUTING, COMPLETED, FAILED, PAUSED, STALE }
```

### Events (`SessionEntity`)

`SessionCreated`, `SessionPlanned`, `ToolDispatched`, `ToolCallBlocked`, `ToolCallRecorded`, `LedgerRevised`, `SessionCompleted`, `SessionFailed`, `SessionPausedOperator`, `SessionFailedTimeout`.

### Events (`BotControlEntity`)

`PauseRequested`, `PauseCleared`.

### Events (`MessageQueue`)

`MessageReceived { sessionId, text, chatId, receivedAt }`.

See `reference/data-model.md` for the full table.

## 6. API contract

Inline surface; see `reference/api-contract.md` for full payload schemas.

- `POST /api/sessions` — body `{ text, chatId? }` → `202 { sessionId }`. Starts a workflow.
- `GET /api/sessions` — list all sessions. Optional `?status=...`.
- `GET /api/sessions/{id}` — one session (full ledgers + reply).
- `GET /api/sessions/sse` — server-sent events stream of every session change.
- `POST /api/control/pause` — body `{ reason }` → `200`. Sets the operator pause flag.
- `POST /api/control/resume` — `200`. Clears the operator pause flag.
- `GET /api/control` — `{ paused, reason }`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — powers the UI tabs.
- `GET /` → redirects to `/app/index.html`.
- `GET /app/*` — static UI.

## 7. UI

The five-tab structure inherited from the exemplar (see `reference/ui-mockup.md`).

- **Overview** — eyebrow "Overview" + headline "Agentic Telegram AI Bot"; no subtitle; cards Try it / How it works / Components / API contract.
- **Architecture** — mermaid diagrams (component graph, sequence, state machine, ER) + click-to-expand component table with hand-tagged syntax-highlighted Java snippets.
- **Risk Survey** — 7 sub-tabs from `governance.html` with answers from `risk-survey.yaml`; unanswered questions faded.
- **Eval Matrix** — 5-column table with click-to-expand rows.
- **App UI** — form to submit a Telegram message, operator pause/resume control, live list of sessions with status pills, expand-row to see the session ledger, the tool ledger entries, and the final reply.

Browser title: `<title>Akka Sample: Agentic Telegram AI Bot</title>`.

The mermaid CSS overrides and `themeVariables` from Lesson 24 must be present. Tab switching must match by `data-tab` / `data-panel` attribute (Lesson 26); no zombie panels left in the DOM.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail** (`before-tool-call` on `RouterAgent`): every `ToolDispatch` is checked against (a) the tool allowlist (`WEB`, `CONTACTS`, `CALENDAR`, `NOTES`), (b) a content policy that forbids NOTES subtasks larger than 2 KB, CONTACTS queries with wildcard-only terms, CALENDAR writes (only reads are permitted), and WEB queries that name off-allowlist hosts. Blocking. Failure → `ToolCallBlocked` entry + replan request.

## 9. Agent prompts

- `RouterAgent` → `prompts/router.md`. Maintains both ledgers; decides next step.
- `WebLookupAgent` → `prompts/web-lookup.md`. Returns search-style answers from fixtures.
- `ContactBookAgent` → `prompts/contact-book.md`. Returns contact records.
- `CalendarQueryAgent` → `prompts/calendar-query.md`. Returns availability and events.
- `NoteSaverAgent` → `prompts/note-saver.md`. Returns save confirmations.

## 10. Acceptance

See `reference/user-journeys.md`. The blueprint generated correctly when:

1. **J1** — Submit "Look up the latest Akka announcements and give me a quick summary." Session progresses `PLANNING → EXECUTING → COMPLETED` within ~3 minutes. UI reflects each transition via SSE. The expanded view shows a session ledger with a non-empty tool plan, a tool ledger with 2–6 entries covering at least `WEB`, and a non-empty `BotReply`.
2. **J2** — Submit a message whose plan would query all contacts with a wildcard. The guardrail blocks the dispatch; the router replans; the session either completes via a revised query or fails after the replan budget is exhausted.
3. **J3** — Submit a message and click **Pause new dispatches** while it is `EXECUTING`. The in-flight tool call finishes; no further dispatches occur; the session ends in `PAUSED`.
4. **J4** — Submit a message whose plan exercises the ContactBook; one fixture response contains an `AKIA...` key shape. The tool ledger entry shows the key replaced by `[REDACTED:aws-access-key]`; the Router's next prompt never contains the literal key.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named telegram-tool-bot demonstrating the
planner-executor × general cell. Runs out of the box (no external services).
Maven group io.akka.samples. Maven artifact planner-executor-general-telegram-tool-bot.
Java package io.akka.samples.agentictelegramaibot. Akka 3.6.0. HTTP port 9368.

Components to wire (exactly):
- 5 AutonomousAgents:
  * RouterAgent — definition() with three capabilities:
      capability(TaskAcceptance.of(PARSE_INTENT).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(DECIDE).maxIterationsPerTask(3)) and
      capability(TaskAcceptance.of(COMPOSE_REPLY).maxIterationsPerTask(2)).
    System prompt from prompts/router.md. PARSE_INTENT returns SessionLedger.
    DECIDE returns a NextStep tagged union (ToolDispatch | Replan | Reply | Fail).
    COMPOSE_REPLY returns BotReply.
  * WebLookupAgent — capability(TaskAcceptance.of(WEB_LOOKUP).maxIterationsPerTask(2)).
    Prompt from prompts/web-lookup.md. Returns ToolResult.
  * ContactBookAgent — capability(TaskAcceptance.of(CONTACT_QUERY).maxIterationsPerTask(2)).
    Prompt from prompts/contact-book.md. Returns ToolResult.
  * CalendarQueryAgent — capability(TaskAcceptance.of(CALENDAR_READ).maxIterationsPerTask(2)).
    Prompt from prompts/calendar-query.md. Returns ToolResult.
  * NoteSaverAgent — capability(TaskAcceptance.of(NOTE_SAVE).maxIterationsPerTask(2)).
    Prompt from prompts/note-saver.md. Returns ToolResult with content holding a
    save-confirmation string.

- 1 Workflow SessionWorkflow with steps:
  planStep -> [loop entry] checkPauseStep -> proposeStep -> guardrailStep ->
  dispatchStep -> sanitizeStep -> recordStep -> decideStep
  -> [back to checkPauseStep or to replyStep / failStep / pausedStep].
  Step timeouts (override settings() per Lesson 4):
    planStep ofSeconds(60), proposeStep ofSeconds(45), dispatchStep
    ofSeconds(120) (covers any tool call), decideStep ofSeconds(45),
    replyStep ofSeconds(60). defaultStepRecovery(maxRetries(2).failoverTo
    (SessionWorkflow::error)).
  checkPauseStep reads BotControlEntity.get; on paused=true transitions to
  pausedStep (emits SessionPausedOperator on SessionEntity).
  guardrailStep runs the deterministic vetter over ToolDispatch; on reject
  records a ToolCallBlocked entry via SessionEntity.recordBlock(subtask, reason)
  and loops back to proposeStep.
  dispatchStep uses switch on ToolDispatch.tool to call the matching agent via
  forAutonomousAgent(...).runSingleTask(...) then forSession(sessionId).result(...).
  sanitizeStep applies SecretScrubber.scrub to the ToolResult.content.
  recordStep calls SessionEntity.recordToolCall(entry).
  decideStep calls forAutonomousAgent(RouterAgent.class, DECIDE);
  on Continue or Replan loops; on Reply transitions to composeReplyStep
  -> replyStep; on Fail transitions to failStep.

- 1 EventSourcedEntity SessionEntity holding Session state. emptyState() returns
  Session.initial("", null) with no commandContext() reference. Commands:
  createSession, recordPlan, recordDispatch, recordBlock, recordToolCall,
  reviseLedger, completeSession, failSession, pauseOperator, timeoutFail,
  getSession. Events as listed in SPEC §5.

- 1 EventSourcedEntity BotControlEntity keyed by literal "global". State
  BotControl{boolean paused, Optional<String> reason, Optional<Instant>
  pausedAt}. Commands: requestPause(reason), clearPause, get. Events:
  PauseRequested, PauseCleared.

- 1 EventSourcedEntity MessageQueue with command enqueueMessage(sessionId,
  text, chatId) emitting MessageReceived.

- 1 View SessionView with row type SessionRow (mirror of Session minus heavy
  ledger payloads — truncate to last 3 tool entries plus counts; the UI fetches
  the full session by id on click). Table updater consumes SessionEntity events.
  ONE query getAllSessions SELECT * AS sessions FROM session_view. No WHERE
  status filter — caller filters client-side (Lesson 2).

- 1 Consumer MessageConsumer subscribed to MessageQueue events; on
  MessageReceived starts a SessionWorkflow with sessionId as the workflow id.

- 2 TimedActions:
  * MessageSimulator — every 75s, reads next line from
    src/main/resources/sample-events/message-prompts.jsonl and calls
    MessageQueue.enqueueMessage.
  * StaleSessionMonitor — every 30s, queries SessionView.getAllSessions,
    filters EXECUTING sessions whose createdAt is older than 4 minutes,
    calls SessionEntity.timeoutFail; SessionWorkflow polls SessionEntity.getSession
    in its decideStep and exits when status == STALE.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions, GET /sessions (filters client-side
    from getAllSessions), GET /sessions/{id}, GET /sessions/sse,
    POST /control/pause, POST /control/resume, GET /control, and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving / -> 302 /app/index.html and /app/* ->
    static-resources/*.

Companion files:
- RouterTasks.java declaring three Task<R> constants: PARSE_INTENT
  (resultConformsTo SessionLedger), DECIDE (NextStep), COMPOSE_REPLY (BotReply).
- ToolExecutorTasks.java declaring four Task<R> constants: WEB_LOOKUP,
  CONTACT_QUERY, CALENDAR_READ, NOTE_SAVE (all resultConformsTo ToolResult).
- Domain records as listed in SPEC §5, plus a NextStep sealed interface with
  permits Continue, Replan, Reply, Fail (each carrying its own payload —
  Continue with ToolDispatch, Replan with revisedLedger, Reply with BotReply,
  Fail with failureReason).
- application/SecretScrubber.java — deterministic regex/entropy scrubber.
  Patterns: AKIA[0-9A-Z]{16}, gh[pousr]_[A-Za-z0-9]{36}, JWT
  ey[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+\.[A-Za-z0-9_-]+, sk-[A-Za-z0-9]{32,},
  Bearer [A-Za-z0-9._-]{20,}, and high-entropy fallback for tokens ≥ 32
  chars whose Shannon entropy > 4.5 bits/char. Replacements:
  [REDACTED:aws-access-key], [REDACTED:github-token],
  [REDACTED:jwt], [REDACTED:openai-key], [REDACTED:bearer-token],
  [REDACTED:high-entropy].
- application/ToolGuardrail.java — deterministic vetter. Reject if the
  tool is not WEB/CONTACTS/CALENDAR/NOTES, if a CONTACTS subtask is a
  bare wildcard query (term is "*" or matches /^\*+$/), if a CALENDAR
  subtask contains write verbs (create|delete|update|cancel), if a NOTES
  subtask content length exceeds 2048 characters, if a WEB subtask names
  a host not on the allowlist (akka.io, doc.akka.io, github.com).
- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port
  = 9368 and akka.javasdk.agent model-provider blocks for anthropic
  (claude-sonnet-4-6), openai (gpt-4o), googleai-gemini (gemini-2.5-flash),
  each api-key read from ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}.
- src/main/resources/sample-events/message-prompts.jsonl with 8 canned
  Telegram-style message prompts spanning web lookup, contact queries,
  calendar reads, and note saves.
- src/main/resources/sample-data/web-fixtures.jsonl — 12 canned web
  fixtures (host, path, title, excerpt). Used by WebLookupAgent.
- src/main/resources/sample-data/contacts.jsonl — 10 contact fixture
  records (name, email, phone, tags). ONE record contains an AKIA-shaped
  field value for the J4 acceptance test.
- src/main/resources/sample-data/calendar-fixtures.jsonl — 8 fixture
  calendar events (eventId, title, start, end, attendees).
- src/main/resources/sample-data/notes-store.jsonl — 4 fixture saved
  notes (noteId, content, savedAt). NoteSaverAgent appends to this at
  runtime; fixture records are read-only stubs for dev.
- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml,
  README.md (copies of the project-root files for the metadata endpoint
  to serve from classpath).
- eval-matrix.yaml at the project root with 1 control (G1) and a matching
  simplified_view list. No regulation_anchors.
- risk-survey.yaml at the project root, pre-filling purpose, data,
  decisions, failure, oversight, operations.external_tool_calls, and
  compliance.capabilities; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.
- prompts/router.md, prompts/web-lookup.md, prompts/contact-book.md,
  prompts/calendar-query.md, prompts/note-saver.md loaded at agent startup
  as system prompts.
- README.md at the project root: title "Akka Sample: Agentic Telegram AI
  Bot", one-line pitch, prerequisites (including the integration form's
  host-software requirement: None), generate-the-system, what-you-get,
  customise-before-generating, what-gets-validated, license. NO
  Configuration section. NO governance-mechanisms section. NO "Visual"
  prefix on tab names.
- src/main/resources/static-resources/index.html — a single self-contained
  HTML file (no ui/ folder, no npm build). Inline CSS + JS. Runtime CDN
  imports for markdown and YAML libs are acceptable. Five tabs matching
  the formal exemplar: Overview, Architecture (4 mermaid diagrams +
  click-to-expand component table with syntax-highlighted Java snippets),
  Risk Survey (7 sub-tabs from governance.html with answers populated from
  risk-survey.yaml; unanswered .qb opacity 0.45), Eval Matrix (5-column
  ID/Control/Mechanism/Implementation/Source table with click-to-expand
  rows), App UI (form + operator pause/resume control + live list with
  status pills and expand-on-click for ledgers and reply). Browser title
  exactly: <title>Akka Sample: Agentic Telegram AI Bot</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:
- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly
  one is set, default application.conf's model-provider to match and
  proceed silently.
- If none is set, ask the user how to source the key, offering five
  options via the conversational surface:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns
        random-but-shape-correct outputs per agent (see Mock LLM provider
        block below for per-agent shapes). Sets model-provider = mock.
    (b) Name an existing env var — record the env-var NAME in
        application.conf; /akka:build forwards the value from the Claude
        session env to the JVM.
    (c) Point to an existing env file — record the PATH in a project-local
        .akka-build.yaml; /akka:build sources the file before spawning the
        JVM.
    (d) Secrets-store URI — 1password://, aws-secretsmanager://, vault://;
        recorded in .akka-build.yaml; resolved at run time.
    (e) Type once in this session — value lives in Claude session memory;
        passed to the JVM via the MCP tool's environment parameter; gone
        when the session ends.
- NEVER write the key value to any file Akka creates. No .env, no entry in
  application.conf, no secrets.yaml, no .akka/ file with key material.
  Akka records only the REFERENCE (env-var name, file path, secrets URI);
  the value lives in the user's existing infrastructure.
- The generated Bootstrap.java fails fast with a clear message naming the
  configured key reference if it does not resolve at runtime. The error
  message must not echo any captured key material.

Mock LLM provider — required when option (a) is selected:
- Generate src/main/java/.../application/MockModelProvider.java implementing
  the ModelProvider interface with per-agent dispatch on the agent class
  name and the Task<R> id. Each branch reads a JSON file from
  src/main/resources/mock-responses/<agent-name>.json (one file per agent
  named in Section 9: router.json, web-lookup.json, contact-book.json,
  calendar-query.json, note-saver.json), picks one entry pseudo-randomly
  per call, and deserialises it into the agent's typed return shape.
- Per-agent mock-response shapes for THIS blueprint:
    router.json — a sectioned file with three lists keyed by task id:
      "PARSE_INTENT" → 4–6 SessionLedger entries (intent strings, facts,
      toolPlan steps that span web/contacts/calendar/notes).
      "DECIDE" → 4–6 NextStep entries covering Continue (with ToolDispatch
      across all four tools), Replan, Reply, Fail. The Continue entries
      advance through a plausible session narrative when iterated in order.
      "COMPOSE_REPLY" → 4–6 BotReply entries with 60–120 word reply texts
      and 2–4 citation bullets.
    web-lookup.json — 6 ToolResult entries, ok=true, content fields are
      mocked search excerpts (4–6 lines each).
    contact-book.json — 6 ToolResult entries; ONE entry's content must
      include the literal substring "AKIAIOSFODNN7EXAMPLE" so the J4
      sanitizer test fires.
    calendar-query.json — 5 ToolResult entries with availability and event
      content.
    note-saver.json — 5 ToolResult entries with save-confirmation content.
- A MockModelProvider.seedFor(sessionId) helper makes the selection
  deterministic per session id so the same session in dev produces the
  same output across restarts.

Constraints — see
explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:
- Lesson 1: AutonomousAgent never silently downgraded to Agent — the
  base class clause must read `extends AutonomousAgent` for every agent.
- Lesson 4: WorkflowSettings.stepTimeout must be set explicitly on every
  step that calls an agent (planStep, proposeStep, dispatchStep,
  decideStep, replyStep).
- Lesson 6: Optional<T> for every nullable field on a View row record and
  on the Session entity state (ledger, toolLedger, reply, failureReason,
  pauseReason, finishedAt).
- Lesson 7: AutonomousAgent requires companion RouterTasks.java and
  ToolExecutorTasks.java declaring every Task<R> constant.
- Lesson 8: model-name values verified against the provider's current
  lineup. Conservative defaults: claude-sonnet-4-6, gpt-4o,
  gemini-2.5-flash.
- Lesson 9: Run command is "/akka:build" (Claude Code), never
  "mvn akka:run".
- Lesson 10: HTTP port 9368 in application.conf — picked from the
  available range.
- Lesson 11: source.platform never appears in any user-facing surface.
- Lesson 12: UI fits 1080px column.
- Lesson 13: integration label is the descriptive string "Runs out of the
  box" — never T1/T2/T3/T4.
- Lesson 23: no competitor brand names in README, SPEC, PLAN, UI, or
  metadata files.
- Lesson 24: static-resources/index.html includes the mermaid CSS
  overrides AND themeVariables (state-diagram label colour, edge-label
  foreignObject overflow:visible, transitionLabelColor #cccccc).
- Lesson 25: API-key sourcing follows the five-option flow above; no key
  value written to disk.
- Lesson 26: Tab switching matches by data-tab / data-panel attribute,
  NEVER by NodeList index. No "hidden" zombie panels in the DOM — delete
  removed tabs, do not display:none them.
- The Overview tab's Try-it card shows just "/akka:build" — not an
  env-var export block.
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
