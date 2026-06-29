# SPEC — ae-oauth

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** OAuthToolAgent.
**One-line pitch:** A caller submits a natural-language request alongside an OAuth token id; one AI agent resolves the token's scope set, then issues tool calls — each of which is validated by a `before-tool-call` guardrail — returning a structured result with a per-call disposition (ALLOWED / DENIED) and the tool output for each permitted call.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the general domain. One `ToolCallerAgent` (AutonomousAgent) handles the entire flow; the surrounding components prepare its token context and record its output. One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** fires on every tool call the agent proposes. It resolves the session's token scope set and checks the proposed tool's required scope against it. If the required scope is absent, the guardrail blocks the call and returns a structured denial to the agent; the agent records the denial and continues with remaining tool calls. If the scope is present, the call proceeds.

The blueprint shows that a governed tool-calling agent does not need post-hoc audit — the enforcement point lives inside the agent loop, not after the fact.

## 3. User-facing flows

The user opens the App UI tab.

1. The user picks a **token** from a dropdown (three seeded tokens with different scope sets: `full-access`, `read-only`, `calendar-only`) or pastes a raw token id.
2. The user types a **request** in a textarea (e.g., "List my calendar events and then create a new event for next Monday at 10 AM").
3. The user clicks **Run**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
4. The session card appears in the live list with status `PENDING`. Within ~1 s it transitions to `TOKEN_RESOLVED` — the resolved scope set is shown in the card detail.
5. Within ~10–30 s the agent finishes. The card transitions to `RUNNING` then `COMPLETED`. The result shows a per-tool-call table: tool name, required scope, disposition chip (ALLOWED / DENIED), and the output or denial reason.
6. The user can submit another request; the live list keeps the history.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `SessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, list, get, SSE; serves `/api/metadata/*`. | — | `SessionEntity`, `SessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `SessionEntity` | `EventSourcedEntity` | Per-session lifecycle: pending → token-resolved → running → completed / failed. Source of truth. | `SessionEndpoint`, `SessionWorkflow` | `SessionView` |
| `SessionWorkflow` | `Workflow` | One workflow per session. Steps: `resolveTokenStep` → `agentStep` → `recordStep`. | started by `SessionEndpoint` after entity submit | `ToolCallerAgent`, `SessionEntity` |
| `ToolCallerAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the request and resolved scope set in the task definition; proposes tool calls that the guardrail validates. Returns `AgentResult`. | invoked by `SessionWorkflow` | returns result |
| `OAuthScopeGuardrail` | (guardrail class) | Validates each proposed tool call against the session's scope set before execution. Blocks and records denied calls. | wired to `ToolCallerAgent` | allows/denies tool calls |
| `SessionView` | `View` | Read model: one row per session for the UI. | `SessionEntity` events | `SessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TokenSpec(
    String tokenId,
    List<String> scopes,
    Instant issuedAt,
    Instant expiresAt
) {}

record SessionRequest(
    String sessionId,
    String tokenId,
    String requestText,
    String submittedBy,
    Instant submittedAt
) {}

record ResolvedToken(
    String tokenId,
    List<String> scopes,
    boolean expired
) {}

record ToolCallRecord(
    String toolName,
    String requiredScope,
    Disposition disposition,   // ALLOWED | DENIED
    String output,             // tool output if ALLOWED, denial reason if DENIED
    Instant calledAt
) {}
enum Disposition { ALLOWED, DENIED }

record AgentResult(
    Outcome outcome,
    String summary,
    List<ToolCallRecord> toolCalls,
    Instant completedAt
) {}
enum Outcome { SUCCESS, PARTIAL, DENIED }

record Session(
    String sessionId,
    Optional<SessionRequest> request,
    Optional<ResolvedToken> token,
    Optional<AgentResult> result,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    PENDING, TOKEN_RESOLVED, RUNNING, COMPLETED, FAILED
}
```

Events on `SessionEntity`: `SessionCreated`, `TokenResolved`, `AgentStarted`, `ResultRecorded`, `SessionFailed`.

Every nullable lifecycle field on the `Session` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ tokenId, requestText, submittedBy }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: OAuth Tool Agent</title>`.

The App UI tab is a two-column layout: a left rail with the submission panel and the live list of sessions (status pill + outcome badge + age) and a right pane with the selected session's detail — resolved scope chips, agent summary, and a per-tool-call table (tool name, required scope, disposition chip, output or denial reason).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every tool call proposed by `ToolCallerAgent`. Fetches the session's resolved `ResolvedToken.scopes`, looks up the proposed tool's `requiredScope` in the tool registry, and checks membership. On scope absence, returns a structured denial `{ disposition: "DENIED", reason: "token lacks scope <scope>" }` to the agent loop; the agent records this in its `toolCalls` list and moves to the next call. On scope presence, allows the call through. Expired tokens are caught in `resolveTokenStep` before the agent is invoked; the session transitions to `FAILED` immediately.

## 9. Agent prompts

- `ToolCallerAgent` → `prompts/tool-caller.md`. The single decision-making LLM. System prompt instructs it to parse the request, plan which tools are needed, propose each call in order, and incorporate both successful outputs and guardrail denials into its final summary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — A `full-access` token submits "list calendar events" → scope `calendar:read` is present → tool call ALLOWED → result appears with `outcome: SUCCESS`.
2. **J2** — A `read-only` token submits "create a new calendar event" → scope `calendar:write` is absent → guardrail blocks → result shows disposition DENIED with reason; `outcome: DENIED`.
3. **J3** — A `calendar-only` token submits a request requiring both `calendar:read` and `files:read` → `calendar:read` ALLOWED, `files:read` DENIED → result shows `outcome: PARTIAL` with mixed disposition row.
4. **J4** — An expired token id is submitted → `resolveTokenStep` detects `expired = true` → `SessionFailed` event → session card shows `FAILED` before any agent call is made.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named ae-oauth demonstrating the single-agent × general cell. Runs
out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-general-oauth-tool-agent. Java package io.akka.samples.aeoauth.
Akka 3.6.0. HTTP port 9138.

Components to wire (exactly):

- 1 AutonomousAgent ToolCallerAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/tool-caller.md>) and
  .capability(TaskAcceptance.of(CALL_TOOLS).maxIterationsPerTask(5)). The task
  receives the request text and resolved scope set in its instruction text; the agent
  proposes tool calls that the OAuthScopeGuardrail validates before execution.
  Output: AgentResult{outcome: Outcome (SUCCESS/PARTIAL/DENIED), summary: String,
  toolCalls: List<ToolCallRecord>, completedAt: Instant}. The agent is configured
  with a before-tool-call guardrail (see G1 in eval-matrix.yaml) registered via the
  agent's guardrail-configuration block.

- 1 Workflow SessionWorkflow per sessionId with three steps:
  * resolveTokenStep — loads the TokenSpec from the in-process token registry
    (TokenRegistry.java) for the submitted tokenId. If the token is expired
    (expiresAt.isBefore(Instant.now())), emits SessionFailed and transitions to FAILED.
    Otherwise emits TokenResolved with the scope list. WorkflowSettings.stepTimeout 5s.
  * agentStep — emits AgentStarted, then calls componentClient.forAutonomousAgent(
    ToolCallerAgent.class, "agent-" + sessionId).runSingleTask(
      TaskDef.instructions(formatRequest(session.request, session.token))
    ) — returns a taskId, then forTask(taskId).result(CALL_TOOLS) to fetch the result.
    On success calls SessionEntity.recordResult(result). WorkflowSettings.stepTimeout 90s
    with defaultStepRecovery maxRetries(1).failoverTo(SessionWorkflow::error).
  * recordStep — calls SessionEntity.complete(). WorkflowSettings.stepTimeout 5s.
    error step calls SessionEntity.fail(reason) and transitions to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity SessionEntity (one per sessionId). State Session{sessionId: String,
  request: Optional<SessionRequest>, token: Optional<ResolvedToken>,
  result: Optional<AgentResult>, status: SessionStatus,
  createdAt: Instant, finishedAt: Optional<Instant>}. SessionStatus enum: PENDING,
  TOKEN_RESOLVED, RUNNING, COMPLETED, FAILED. Events: SessionCreated{request},
  TokenResolved{token}, AgentStarted{}, ResultRecorded{result}, SessionFailed{reason}.
  Commands: create, resolveToken, markRunning, recordResult, complete, fail, getSession.
  emptyState() returns Session.initial("") with no commandContext() reference (Lesson 3).
  Every Optional<T> field uses Optional.empty() in initial state and Optional.of(...)
  inside the event-applier.

- 1 View SessionView with row type SessionRow (mirrors Session). Table updater consumes
  SessionEntity events. ONE query getAllSessions: SELECT * AS sessions FROM session_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * SessionEndpoint at /api with POST /sessions (body {tokenId, requestText, submittedBy};
    mints sessionId; calls SessionEntity.create; starts SessionWorkflow; returns {sessionId}),
    GET /sessions (list from getAllSessions, sorted newest-first), GET /sessions/{id}
    (one row), GET /sessions/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- SessionTasks.java declaring one Task<R> constant: CALL_TOOLS = Task.name("Call tools")
  .description("Parse the request, check OAuth scopes via the guardrail, and return an
  AgentResult with one ToolCallRecord per attempted call")
  .resultConformsTo(AgentResult.class). DO NOT skip this — the AutonomousAgent requires
  its companion Tasks class (Lesson 7).

- Domain records TokenSpec, SessionRequest, ResolvedToken, ToolCallRecord, Disposition,
  AgentResult, Outcome, Session, SessionStatus.

- OAuthScopeGuardrail.java implementing the before-tool-call hook. On each proposed tool
  call it (1) looks up the tool name in ToolRegistry.TOOL_SCOPES to find the requiredScope,
  (2) checks that requiredScope is present in the session's resolved token scopes,
  (3) if absent returns Guardrail.deny({disposition:"DENIED", reason:"token lacks scope
  <scope>"}) so the agent records the denial and continues. If present, returns
  Guardrail.allow() so the tool executes.

- TokenRegistry.java — immutable in-process map of seeded token ids to TokenSpec objects.
  Three seeded tokens: "full-access" (scopes: calendar:read, calendar:write, files:read,
  files:write, contacts:read, expires 2099-01-01), "read-only" (scopes: calendar:read,
  files:read, contacts:read, expires 2099-01-01), "calendar-only" (scopes: calendar:read,
  calendar:write, expires 2099-01-01). Also includes one expired token "expired-token"
  (scopes: calendar:read, expiresAt: 2020-01-01) for J4 testing.

- ToolRegistry.java — immutable in-process map of tool names to their requiredScope. Tools:
  "listCalendarEvents" → "calendar:read", "createCalendarEvent" → "calendar:write",
  "listFiles" → "files:read", "uploadFile" → "files:write", "listContacts" → "contacts:read".

- SimulatedToolExecutor.java — executes permitted tool calls with deterministic in-process
  responses (no external API calls). One response per tool: listCalendarEvents returns a
  JSON array of 3 synthetic calendar events; createCalendarEvent returns a synthetic event
  id; listFiles returns 5 synthetic file entries; uploadFile returns a synthetic upload
  confirmation; listContacts returns 4 synthetic contacts.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9138 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/tokens.jsonl with the four seeded TokenSpec records.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with data.data_classes.pii = false (token ids are
  opaque, no user PII in the request flow), decisions.authority_level = automated (the
  guardrail's allow/deny is enforced, not advisory), oversight.human_in_loop = false
  (the scope check is deterministic and does not require human confirmation),
  failure.failure_modes including "scope-bypass", "token-replay", "over-privileged-scope",
  "guardrail-misconfiguration"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/tool-caller.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: OAuth Tool Agent", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left
  = submission panel + live list of session cards; right = selected-session detail with
  resolved scope chips, agent summary, and per-tool-call table).
  Browser title exactly: <title>Akka Sample: OAuth Tool Agent</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct AgentResult values per task (see Mock LLM provider block below). Sets
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
- NEVER write the key value to any file Akka creates.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    call-tools.json — 6 AgentResult entries covering the three Outcome values.
      Each entry has a summary and a toolCalls array with realistic ToolCallRecord entries.
      Entries include full-success (all ALLOWED), full-denial (all DENIED), and partial
      (mixed). The guardrail in the agent loop still validates each call even in mock mode —
      the mock's proposed calls must match the ToolRegistry names so the guardrail has real
      scope data to check.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ToolCallerAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion SessionTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (resolveTokenStep 5s, agentStep
  90s, recordStep 5s, error 5s).
- Lesson 6: every nullable lifecycle field on the Session row record is Optional<T>.
- Lesson 7: SessionTasks.java with CALL_TOOLS = Task.name(...).description(...)
  .resultConformsTo(AgentResult.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9138 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4 in any user-visible
  string.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes the mermaid CSS overrides
  and mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ToolCallerAgent).
  OAuthScopeGuardrail is a guardrail class, not a second agent.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external post-hoc check.
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
