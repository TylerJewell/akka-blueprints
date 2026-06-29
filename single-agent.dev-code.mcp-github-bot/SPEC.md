# SPEC — mcp-github-bot

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** MCP GitHub Bot.
**One-line pitch:** A user types a natural-language request; one AI agent translates it into GitHub MCP tool calls — listing repos, reading and creating issues, adding comments — while a pre-tool guardrail blocks unsafe or halted write operations before they reach GitHub.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the dev-code domain. One `GitHubBotAgent` (AutonomousAgent) carries the entire decision about which GitHub MCP tools to invoke; surrounding components protect that tool surface with two governance controls:

- A **before-tool-call guardrail** runs inside the agent's loop on every pending tool call. It checks the tool name against an allowlist of permitted GitHub operations and reads a runtime halt flag. If a write-class tool is requested while the halt flag is set, the guardrail rejects the call and the agent must explain the denial to the user without making any GitHub mutation.
- A **halt mechanism** lets an operator or automated regulator toggle a `writeHalted` flag on `HaltFlagEntity` at runtime — no redeployment, no code change. The guardrail reads this flag on every write-class dispatch.

The blueprint demonstrates that even when the agent's tool surface is external (a live MCP server), strong governance is achievable: the guardrail sits between the agent's intent and the network call.

## 3. User-facing flows

The user opens the App UI tab.

1. The user types a request into the **Request** textarea (or picks one of four seeded examples: list repositories, read open issues, create an issue, add a comment to an issue).
2. The user fills in **Repository** (owner/repo format) and their **GitHub token** (one-time entry, never stored).
3. The user clicks **Send request**. The UI POSTs to `/api/sessions` and receives a `sessionId`.
4. The session card appears in the live list in `SUBMITTED` state. Within ~1 s it transitions to `RUNNING`.
5. Within ~10–30 s the agent returns. The card transitions to `COMPLETED` (or `FAILED` if the agent errored or all write calls were halted). The right pane shows: the agent's natural-language response, the ordered list of tool calls it made (tool name, inputs, outcome), and any guardrail-blocked calls with the rejection reason.
6. An operator button at the top of the UI toggles the write-halt flag. When active, the flag chip shows `WRITES HALTED` in red; when inactive it shows `WRITES ACTIVE` in green.
7. The user can submit another request; the live list keeps the history visible, newest-first.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `BotSessionEndpoint` | `HttpEndpoint` | `/api/sessions/*` — submit, list, get, SSE; `/api/halt` — read/toggle halt flag; `/api/metadata/*`. | — | `BotSessionEntity`, `HaltFlagEntity`, `BotSessionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `BotSessionEntity` | `EventSourcedEntity` | Per-session lifecycle: submitted → running → completed / failed. Source of truth for session state and tool-call log. | `BotSessionEndpoint`, `BotSessionWorkflow` | `BotSessionView` |
| `HaltFlagEntity` | `EventSourcedEntity` | Singleton (id `"global"`) holding the boolean `writeHalted` flag. Commands: `enableHalt`, `disableHalt`, `getFlag`. | `BotSessionEndpoint` (operator action) | `ToolCallGuardrail` reads it |
| `BotSessionWorkflow` | `Workflow` | One workflow per session. Steps: `runAgentStep` → `recordStep`. | started by `BotSessionEndpoint` after submit | `GitHubBotAgent`, `BotSessionEntity` |
| `GitHubBotAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the user request and GitHub context in the task definition; invokes GitHub MCP tools; returns a `BotResponse`. | invoked by `BotSessionWorkflow` | returns `BotResponse` |
| `ToolCallGuardrail` | (guardrail hook) | `before-tool-call` hook registered on `GitHubBotAgent`. Reads `HaltFlagEntity.getFlag` and validates tool name against an allowlist. Returns structured rejection or pass-through. | `GitHubBotAgent` tool-call loop | `HaltFlagEntity` |
| `BotSessionView` | `View` | Read model: one row per session for the UI. | `BotSessionEntity` events | `BotSessionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record BotRequest(
    String sessionId,
    String userRequest,
    String repository,       // owner/repo
    String githubToken,      // passed into MCP server config; never persisted in entity state
    String submittedBy,
    Instant submittedAt
) {}

record ToolCallRecord(
    String toolName,
    String inputSummary,     // JSON-serialized inputs, truncated at 500 chars
    String outcome,          // "success", "error", "blocked"
    String resultSummary,    // first 500 chars of tool result or rejection reason
    Instant calledAt
) {}

record BotResponse(
    String agentMessage,                // natural-language summary from the agent
    List<ToolCallRecord> toolCalls,     // ordered list of every tool call attempted
    int blockedCallCount,               // how many calls the guardrail blocked
    Instant respondedAt
) {}

record BotSession(
    String sessionId,
    Optional<BotRequest> request,
    Optional<BotResponse> response,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    SUBMITTED, RUNNING, COMPLETED, FAILED
}

record HaltFlag(
    boolean writeHalted,
    Optional<String> haltedBy,
    Optional<Instant> haltedAt
) {}
```

Events on `BotSessionEntity`: `SessionSubmitted`, `SessionStarted`, `SessionCompleted`, `SessionFailed`.
Events on `HaltFlagEntity`: `WriteHaltEnabled`, `WriteHaltDisabled`.

Every nullable lifecycle field on `BotSession` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/sessions` — body `{ userRequest, repository, githubToken, submittedBy }` → `{ sessionId }`.
- `GET /api/sessions` — list all sessions, newest-first.
- `GET /api/sessions/{id}` — one session.
- `GET /api/sessions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/halt` — current halt flag state `{ writeHalted, haltedBy, haltedAt }`.
- `POST /api/halt/enable` — operator sets `writeHalted = true`.
- `POST /api/halt/disable` — operator sets `writeHalted = false`.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: MCP GitHub Bot</title>`.

The App UI tab is a two-column layout: a left rail with the halt-flag toggle at the top, a request form below it, and a live list of sessions (status pill + session age); a right pane with the selected session's detail — user request, tool-call log (tool name, inputs, outcome chips), guardrail-blocked calls in red, and the agent's natural-language response.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs on every pending MCP tool call inside `GitHubBotAgent`'s loop. Checks two conditions: (1) the tool name is in the configured allowlist of permitted GitHub operations (`list_repos`, `get_issue`, `list_issues`, `create_issue`, `add_comment`); (2) if the tool is a write-class operation (`create_issue`, `add_comment`), reads `HaltFlagEntity.getFlag` and rejects if `writeHalted == true`. Returns a structured rejection carrying the blocked tool name and the blocking reason so the agent can relay the denial to the user. Passing calls flow through to the MCP server.
- **H1 — operator-regulator halt**: `HaltFlagEntity` holds a singleton boolean flag. `POST /api/halt/enable` flips it to `true`; `POST /api/halt/disable` flips it back. The UI exposes both as an operator toggle button. No redeployment is needed to halt or resume write access.

## 9. Agent prompts

- `GitHubBotAgent` → `prompts/github-bot.md`. The single decision-making LLM. System prompt instructs it to translate the user's natural-language request into GitHub MCP tool calls, record every tool call in its response, and explain any guardrail-blocked calls rather than silently omitting them.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User submits "list open issues in owner/repo"; within 30 s the agent returns a list of issues; the tool-call log shows `list_issues` with a `success` outcome.
2. **J2** — Operator sets halt flag; user submits "create an issue titled Test"; the guardrail blocks `create_issue`; the agent's response explains the write block; no GitHub mutation occurs.
3. **J3** — Operator disables halt flag; same user submits "create an issue titled Test" again; the tool call succeeds; `blockedCallCount == 0`.
4. **J4** — User submits a request invoking a tool outside the allowlist; guardrail blocks with `tool-not-permitted`; agent relays the limitation without erroring.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named mcp-github-bot demonstrating the single-agent × dev-code cell. Connects
to a GitHub MCP server (read+write). Maven group io.akka.samples. Maven artifact
single-agent-dev-code-mcp-github-bot. Java package io.akka.samples.mcpgithubbot. Akka 3.6.0.
HTTP port 9869.

Components to wire (exactly):

- 1 AutonomousAgent GitHubBotAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/github-bot.md>),
  .capability(TaskAcceptance.of(EXECUTE_GITHUB_REQUEST).maxIterationsPerTask(5)), and
  .mcpServer("github", McpServerConfig.fromEnv("GITHUB_TOKEN")) pointing at the GitHub MCP
  server (https://github.com/modelcontextprotocol/servers/tree/main/src/github). Output:
  BotResponse{agentMessage: String, toolCalls: List<ToolCallRecord>, blockedCallCount: int,
  respondedAt: Instant}. The agent is configured with a before-tool-call guardrail registered
  via the agent's guardrail-configuration block (see G1 in eval-matrix.yaml).

- 1 Workflow BotSessionWorkflow per sessionId with two steps:
  * runAgentStep — emits SessionStarted, then calls componentClient.forAutonomousAgent(
    GitHubBotAgent.class, "bot-" + sessionId).runSingleTask(
      TaskDef.instructions(formatRequest(session.request))
    ) — returns a taskId, then forTask(taskId).result(EXECUTE_GITHUB_REQUEST) to fetch
    BotResponse. WorkflowSettings.stepTimeout 90s with defaultStepRecovery
    maxRetries(1).failoverTo(BotSessionWorkflow::error).
  * recordStep — calls BotSessionEntity.complete(response). WorkflowSettings.stepTimeout 10s.
    error step calls BotSessionEntity.fail(reason). WorkflowSettings.stepTimeout 5s.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 2 EventSourcedEntities:

  * BotSessionEntity (one per sessionId). State BotSession{sessionId: String, request:
    Optional<BotRequest>, response: Optional<BotResponse>, status: SessionStatus,
    createdAt: Instant, finishedAt: Optional<Instant>}. SessionStatus enum: SUBMITTED,
    RUNNING, COMPLETED, FAILED. Events: SessionSubmitted{request (without githubToken —
    never persist the token)}, SessionStarted{}, SessionCompleted{response},
    SessionFailed{reason}. Commands: submit, start, complete, fail, getSession.
    emptyState() returns BotSession.initial("") with Optional.empty() fields and
    status = SUBMITTED (Lesson 3).

  * HaltFlagEntity (singleton, id "global"). State HaltFlag{writeHalted: boolean,
    haltedBy: Optional<String>, haltedAt: Optional<Instant>}. Events: WriteHaltEnabled
    {enabledBy: String, enabledAt: Instant}, WriteHaltDisabled{disabledBy: String,
    disabledAt: Instant}. Commands: enableHalt(operator: String), disableHalt(operator:
    String), getFlag. emptyState() returns HaltFlag{writeHalted: false,
    haltedBy: Optional.empty(), haltedAt: Optional.empty()} (Lesson 3).

- 1 Consumer — NOT needed. BotSessionWorkflow is started directly by BotSessionEndpoint
  after the submit response. No Consumer subscription required.

- 1 View BotSessionView with row type BotSessionRow (mirrors BotSession; includes
  sessionId, status, createdAt, finishedAt, agentMessage from response if present,
  blockedCallCount from response if present). Table updater consumes BotSessionEntity
  events. ONE query getAllSessions: SELECT * AS sessions FROM bot_session_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller
  filters client-side.

- 2 HttpEndpoints:
  * BotSessionEndpoint at /api with:
    POST /sessions (body {userRequest, repository, githubToken, submittedBy}; mints
      sessionId via UUID; calls BotSessionEntity.submit (passing request WITHOUT
      githubToken in the entity command — the token is forwarded only to the workflow
      start call); starts BotSessionWorkflow; returns {sessionId}),
    GET /sessions (list from getAllSessions, sorted newest-first),
    GET /sessions/{id} (one row),
    GET /sessions/sse (Server-Sent Events forwarded from the view's stream-updates),
    GET /halt (reads HaltFlagEntity.getFlag),
    POST /halt/enable (calls HaltFlagEntity.enableHalt; body {operator: String}),
    POST /halt/disable (calls HaltFlagEntity.disableHalt; body {operator: String}),
    GET /metadata/readme, GET /metadata/risk-survey, GET /metadata/eval-matrix (serve
      YAML/MD from src/main/resources/metadata/).
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- BotTasks.java declaring one Task<R> constant: EXECUTE_GITHUB_REQUEST = Task
  .name("Execute GitHub request")
  .description("Translate the natural-language request into GitHub MCP tool calls and return a BotResponse")
  .resultConformsTo(BotResponse.class). DO NOT skip this — the AutonomousAgent
  requires its companion Tasks class (Lesson 7).

- Domain records BotRequest, ToolCallRecord, BotResponse, BotSession, SessionStatus,
  HaltFlag.

- ToolCallGuardrail.java implementing the before-tool-call hook. On each pending tool call:
  (1) checks the tool name is in PERMITTED_TOOLS = {list_repos, get_issue, list_issues,
  create_issue, add_comment}; (2) if the tool is write-class (create_issue, add_comment),
  reads HaltFlagEntity via componentClient.forEventSourcedEntity(HaltFlagEntity.class,
  "global").call(HaltFlagEntity::getFlag) and rejects if writeHalted == true. On rejection
  returns Guardrail.reject(structured-error containing toolName + blockingReason). On pass
  returns Guardrail.allow().

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9869 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}. The GitHubBotAgent.definition() binds the configured
  provider via the per-agent override pattern from the akka-context docs. GitHub MCP server
  credentials are sourced from ${?GITHUB_TOKEN} at runtime — never stored in application.conf
  as a literal value.

- src/main/resources/sample-events/seed-requests.jsonl with 4 seeded request examples:
  (1) "List all open issues in owner/repo", (2) "Show me the latest 5 repositories for
  owner", (3) "Create an issue titled 'Automated governance test' in owner/repo",
  (4) "Add a comment 'Acknowledged' to issue #42 in owner/repo".

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching Section 8.
  Matching simplified_view list.

- risk-survey.yaml at the project root with data.data_classes.pii = false,
  decisions.authority_level = autonomous-action (the agent mutates external state),
  oversight.human_in_loop = false, oversight.human_on_loop = true (operator monitors and
  can halt), failure.failure_modes including "unintended-issue-creation",
  "runaway-comment-spam", "token-exposure-in-logs", "tool-not-in-allowlist";
  deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/github-bot.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: MCP GitHub Bot", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  halt-flag toggle at top + request form + live session list; right = selected-session detail
  with user request, tool-call log, blocked calls, and agent response).
  Browser title exactly: <title>Akka Sample: MCP GitHub Bot</title>. No subtitle on the
  Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- Also check GITHUB_TOKEN. If set, record the env-var name in the McpServerConfig and
  proceed silently. If not set, offer the same five options as for the model key (including
  mock — in mock mode the GitHub MCP tool calls are stubbed with seed responses from
  src/main/resources/mock-responses/).
- If neither model key nor GitHub token is set, ask once covering both.
- NEVER write either key value to any file Akka creates.

Mock LLM / Mock MCP provider — required when option (a) is selected:

- Generate MockModelProvider.java and MockGitHubMcpServer.java (a local stub that returns
  deterministic JSON responses for each tool name from src/main/resources/mock-responses/).
- Per-tool mock-response shapes:
    list_repos.json — 3 repos per call.
    list_issues.json — 5 open issues per call.
    get_issue.json — 1 issue detail.
    create_issue.json — created issue with number and URL.
    add_comment.json — comment id and body.
  Plus 1 entry in create_issue.json marked as triggering the halt path (selected on every
  2nd session in mock mode) so J2 is reproducible without a live halt toggle.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. GitHubBotAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion BotTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (runAgentStep 90s, recordStep
  10s, error 5s).
- Lesson 6: every nullable lifecycle field on BotSession is Optional<T>.
- Lesson 7: BotTasks.java with EXECUTE_GITHUB_REQUEST = Task.name(...)
  .description(...).resultConformsTo(BotResponse.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9869 declared explicitly in application.conf's
  akka.javasdk.dev-mode.http-port.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label is "Requires GitHub credential" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes mermaid CSS overrides
  (state-label colour, edge-label foreignObject overflow:visible) AND the
  mermaid.initialize themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab/data-panel attribute only; exactly five
  <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (GitHubBotAgent). HaltFlagEntity
  is a plain EventSourcedEntity with no LLM involvement.
- The GitHub token is forwarded to the MCP server config only; it MUST NOT appear in any
  persisted entity state, view row, or HTTP response body.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration mechanism,
  not as an external check. It must read the live HaltFlagEntity state on each call.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks. Same default-everything policy.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key or GitHub token (offer the three valid model-key env vars and `GITHUB_TOKEN`; the project-local `.akka-build.yaml` written by `/akka:specify` per Section 11 should normally cover this), an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
