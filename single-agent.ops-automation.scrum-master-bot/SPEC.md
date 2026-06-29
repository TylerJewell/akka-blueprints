# SPEC — scrum-master-bot

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** ScrumMasterBot.
**One-line pitch:** A Scrum Master agent conducts daily standups for a sprint team, records each member's progress and blockers, and posts a structured standup summary back to the appropriate tickets — all ticket writes gated by a scope-checked `before-tool-call` guardrail.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `ScrumMasterAgent` (AutonomousAgent) carries the entire standup; the surrounding components prepare its input (sprint state, member roster) and gate its output (ticket writes). One governance mechanism is wired around the agent:

- A **before-tool-call guardrail** runs before every `postTicketUpdate` tool invocation inside the agent. It verifies that the target ticket id is within the current sprint's authorized ticket set. A write to an out-of-scope ticket is rejected with a structured error; the agent must select a valid target or skip the write.

The blueprint shows that a single-agent ops system is not inherently ungoverned — a targeted guardrail on the one external-effect call is enough to prevent the agent from touching tickets it was never authorized to modify.

## 3. User-facing flows

The user opens the App UI tab.

1. The user sees the current sprint fixture loaded (team name, sprint number, member roster, ticket list). They can edit the sprint fixture inline or load one of three seeded examples (a 3-person feature team, a 4-person platform team, a 5-person release-management team).
2. The user clicks **Run standup**. The UI POSTs to `/api/standups` and receives a `sessionId`.
3. The session card appears in the live list in `COLLECTING` state. Within ~1 s it transitions to `RUNNING` — the agent is conducting the standup.
4. Within ~15–60 s the agent finishes. The card transitions to `SUMMARY_READY`. The standup summary appears: a top-level `sessionOutcome` badge (`ON_TRACK` / `AT_RISK` / `BLOCKED`), a per-member row (member id, status, yesterday, today, blocker if any), and a `nextActions` list.
5. Within ~1 s of `SUMMARY_READY`, the `postStep` finishes. The card transitions to `POSTED`. Each ticket in the summary's `ticketUpdates` list shows a posted-to badge.
6. The user can start another standup; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `StandupEndpoint` | `HttpEndpoint` | `/api/standups/*` — start, list, get, SSE; serves `/api/metadata/*`. | — | `StandupEntity`, `StandupView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `StandupEntity` | `EventSourcedEntity` | Per-session lifecycle: collecting → running → summary-ready → posted. Source of truth. | `StandupEndpoint`, `SprintConsumer`, `StandupWorkflow` | `StandupView` |
| `SprintConsumer` | `Consumer` | Subscribes to `SprintActivated` events; loads sprint roster and ticket list; calls `StandupEntity.attachSprintContext`. | `StandupEntity` events | `StandupEntity` |
| `StandupWorkflow` | `Workflow` | One workflow per session. Steps: `collectSprintStep` → `standupStep` → `postStep`. | started by `StandupEndpoint` | `ScrumMasterAgent`, `StandupEntity` |
| `ScrumMasterAgent` | `AutonomousAgent` | The one decision-making LLM. Receives the sprint context (roster + tickets) in the task definition; returns `StandupSummary`. Wired with `TicketWriteGuardrail` on `before-tool-call`. | invoked by `StandupWorkflow` | returns summary |
| `TicketWriteGuardrail` | supporting class | Before every `postTicketUpdate` tool call, asserts the target ticket id is in the sprint's authorized ticket set. | invoked by agent loop | blocks or passes call |
| `StandupView` | `View` | Read model: one row per session for the UI. | `StandupEntity` events | `StandupEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record TeamMember(String memberId, String displayName, String role) {}

record SprintContext(
    String sprintId,
    String sprintName,
    int sprintNumber,
    String teamName,
    List<TeamMember> members,
    List<String> authorizedTicketIds,
    Instant sprintStartAt,
    Instant sprintEndAt
) {}

record MemberUpdate(
    String memberId,
    String yesterday,
    String today,
    Optional<String> blocker
) {}

record TicketUpdate(
    String ticketId,
    String comment,
    Optional<String> newStatus
) {}

record StandupSummary(
    SessionOutcome sessionOutcome,
    String summaryText,
    List<MemberUpdate> memberUpdates,
    List<TicketUpdate> ticketUpdates,
    List<String> nextActions,
    Instant conductedAt
) {}
enum SessionOutcome { ON_TRACK, AT_RISK, BLOCKED }

record PostResult(
    int ticketsPosted,
    List<String> skippedTicketIds,
    Instant postedAt
) {}

record StandupSession(
    String sessionId,
    Optional<SprintContext> sprintContext,
    Optional<StandupSummary> summary,
    Optional<PostResult> postResult,
    SessionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum SessionStatus {
    COLLECTING, RUNNING, SUMMARY_READY, POSTED, FAILED
}
```

Events on `StandupEntity`: `SprintActivated`, `SprintContextAttached`, `StandupStarted`, `SummaryRecorded`, `UpdatesPosted`, `SessionFailed`.

Every nullable lifecycle field on `StandupSession` is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/standups` — body `{ sprintId, teamName, members: [TeamMember], authorizedTicketIds: [String] }` → `{ sessionId }`.
- `GET /api/standups` — list all sessions, newest-first.
- `GET /api/standups/{id}` — one session.
- `GET /api/standups/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: ScrumMasterBot</title>`.

The App UI tab is a two-column layout: a left rail with the live list of standup sessions (status pill + outcome badge + age) and a right pane with the selected session's detail — sprint context (sprint name, team name, roster), standup summary, per-member update rows, ticket updates with posted badges, and next-actions list.

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs before every `postTicketUpdate` tool invocation inside `ScrumMasterAgent`. Asserts the `ticketId` argument is present in `sprintContext.authorizedTicketIds`. If the ticket id is absent from the authorized set, returns a structured `out-of-scope-ticket` rejection to the agent loop, which must then select a valid ticket or omit the write. Passing calls proceed to the simulated ticket tool.

## 9. Agent prompts

- `ScrumMasterAgent` → `prompts/scrum-master.md`. The single decision-making LLM. System prompt instructs it to conduct a standup by asking each member the three standup questions (what did you do yesterday, what will you do today, do you have any blockers?), compile the answers into a `StandupSummary`, and call `postTicketUpdate` once per ticket update in the summary.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — User activates the feature-team sprint fixture; within 60 s the standup summary appears with one `MemberUpdate` per team member and `POSTED` status.
2. **J2** — The agent attempts to post to a ticket not in `authorizedTicketIds` — the `before-tool-call` guardrail rejects it; the agent logs the rejection and moves on; only in-scope tickets are posted.
3. **J3** — All three team members report a blocker → `sessionOutcome` is `BLOCKED` and the live list card shows a red outcome badge.
4. **J4** — A new sprint fixture is loaded mid-use; the next standup session uses the updated roster and ticket scope.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named scrum-master-bot demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-scrum-master-bot. Java package
io.akka.samples.scrummasterassistant. Akka 3.6.0. HTTP port 9181.

Components to wire (exactly):

- 1 AutonomousAgent ScrumMasterAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/scrum-master.md>) and
  .capability(TaskAcceptance.of(CONDUCT_STANDUP).maxIterationsPerTask(4)). The task receives
  the sprint context (roster + authorized ticket list) as its instruction text. Output:
  StandupSummary{sessionOutcome: SessionOutcome (ON_TRACK/AT_RISK/BLOCKED), summaryText:
  String, memberUpdates: List<MemberUpdate>, ticketUpdates: List<TicketUpdate>,
  nextActions: List<String>, conductedAt: Instant}. The agent is configured with a
  before-tool-call guardrail (see G1 in eval-matrix.yaml) via the agent's
  guardrail-configuration block. On guardrail rejection the agent loop skips the write and
  continues within its 4-iteration budget.

- 1 Workflow StandupWorkflow per sessionId with three steps:
  * collectSprintStep — polls StandupEntity.getSession every 1s; on
    session.sprintContext().isPresent() advances to standupStep.
    WorkflowSettings.stepTimeout 15s.
  * standupStep — emits StandupStarted, then calls componentClient.forAutonomousAgent(
    ScrumMasterAgent.class, "scrum-" + sessionId).runSingleTask(
      TaskDef.instructions(formatSprintContext(session.sprintContext()))
    ) — returns a taskId, then forTask(taskId).result(CONDUCT_STANDUP) to fetch the summary.
    On success calls StandupEntity.recordSummary(summary). WorkflowSettings.stepTimeout 120s
    with defaultStepRecovery maxRetries(2).failoverTo(StandupWorkflow::error).
  * postStep — iterates summary.ticketUpdates(), calls the simulated
    TicketPostingService.post(ticketId, comment, newStatus) for each, collects results into
    PostResult, calls StandupEntity.recordPostResult(postResult). WorkflowSettings.stepTimeout
    30s. error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity StandupEntity (one per sessionId). State StandupSession{sessionId:
  String, sprintContext: Optional<SprintContext>, summary: Optional<StandupSummary>,
  postResult: Optional<PostResult>, status: SessionStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. SessionStatus enum: COLLECTING, RUNNING, SUMMARY_READY,
  POSTED, FAILED. Events: SprintActivated{sprintId, teamName, members, authorizedTicketIds},
  SprintContextAttached{sprintContext}, StandupStarted{}, SummaryRecorded{summary},
  UpdatesPosted{postResult}, SessionFailed{reason}. Commands: activate, attachSprintContext,
  markRunning, recordSummary, recordPostResult, fail, getSession. emptyState() returns
  StandupSession.initial("") with no commandContext() reference (Lesson 3). Every
  Optional<T> field uses Optional.empty() in initial state and Optional.of(...) inside the
  event-applier.

- 1 Consumer SprintConsumer subscribed to StandupEntity events; on SprintActivated builds a
  SprintContext from the event payload (sprint number, dates, etc.) and calls
  StandupEntity.attachSprintContext(context). After attachSprintContext lands, the same
  Consumer starts a StandupWorkflow with id = "standup-" + sessionId.

- 1 View StandupView with row type SessionRow (mirrors StandupSession). Table updater
  consumes StandupEntity events. ONE query getAllSessions: SELECT * AS sessions FROM
  standup_view. No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2);
  caller filters client-side.

- 2 HttpEndpoints:
  * StandupEndpoint at /api with POST /standups (body
    {sprintId, teamName, members: [{memberId, displayName, role}], authorizedTicketIds:
    [String]}; mints sessionId; calls StandupEntity.activate; returns {sessionId}),
    GET /standups (list from getAllSessions, sorted newest-first), GET /standups/{id}
    (one row), GET /standups/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- StandupTasks.java declaring one Task<R> constant: CONDUCT_STANDUP = Task.name("Conduct
  standup").description("Run the daily standup for the sprint team and return a
  StandupSummary").resultConformsTo(StandupSummary.class). DO NOT skip this — the
  AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records TeamMember, SprintContext, MemberUpdate, TicketUpdate, StandupSummary,
  SessionOutcome, PostResult, StandupSession, SessionStatus.

- TicketWriteGuardrail.java implementing the before-tool-call hook. Before every
  postTicketUpdate call, extracts the ticketId argument from the tool-call payload, checks it
  against session.sprintContext().authorizedTicketIds(), and either passes the call through
  or returns Guardrail.reject("out-of-scope-ticket: " + ticketId) to prevent the write.

- TicketPostingService.java — a simulated ticket client (no real API). Inputs: ticketId,
  comment, optional newStatus. Outputs: boolean posted. Implementation just logs the call
  and returns true, simulating a successful post. Javadoc explains how to replace it with a
  real HTTP client.

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9181 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}. The ScrumMasterAgent.definition()
  binds the configured provider via .modelProvider("${akka.javasdk.agent.default}") or the
  per-agent override pattern from the akka-context docs.

- src/main/resources/sample-events/sprint-fixtures.jsonl with 3 seeded sprint fixtures:
  a 3-member feature team (sprint 12, 6 tickets), a 4-member platform team (sprint 5,
  8 tickets), and a 5-member release-management team (sprint 3, 10 tickets).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of the
  root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 1 control (G1) matching the mechanism in
  Section 8 of this SPEC. Matching simplified_view list. No regulation_anchors.

- risk-survey.yaml at the project root with purpose.primary_function = standup-automation,
  decisions.authority_level = automated-with-scope-gate (the agent posts ticket updates
  autonomously but within a guardrail-enforced scope), oversight.human_in_loop = false (the
  standup runs unattended; a human reviews the summary after), failure.failure_modes
  including "out-of-scope-ticket-write", "missed-blocker", "stale-sprint-context",
  "hallucinated-member-update"; deployer fields marked TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/scrum-master.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Scrum Master Assistant",
  prerequisites, generate-the-system, what-you-get, customise-before-generating,
  what-gets-validated, license. NO Configuration section. NO governance-mechanisms section
  (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout (left =
  live list of standup session cards; right = selected-session detail with sprint context,
  member update rows, ticket update badges, summary text, next-actions list, and outcome
  badge). Browser title exactly: <title>Akka Sample: ScrumMasterBot</title>. No subtitle
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

- Generate MockModelProvider.java implementing the ModelProvider interface with a per-task
  dispatch on the Task<R> id. Each branch reads src/main/resources/mock-responses/
  <task-id>.json, picks one entry pseudo-randomly per call (seedFor(sessionId)), and
  deserialises into the task's typed return.
- Per-task mock-response shapes for THIS blueprint:
    conduct-standup.json — 6 StandupSummary entries covering all three SessionOutcome values
      (ON_TRACK x2, AT_RISK x2, BLOCKED x2). Each entry has a summaryText paragraph, a
      memberUpdates list with one MemberUpdate per team member in the fixture (yesterday,
      today, optional blocker), a ticketUpdates list with 2–4 TicketUpdate entries, and a
      nextActions list of 2–3 items. Plus 2 deliberately out-of-scope entries whose
      ticketUpdates reference ticket ids not in the sprint's authorizedTicketIds — the
      guardrail blocks those writes, exercising J2.
- A MockModelProvider.seedFor(sessionId) helper makes per-session selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. ScrumMasterAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion StandupTasks.java MUST exist.
- Lesson 4: every workflow step that calls an agent has an explicit stepTimeout (standupStep
  120s, collectSprintStep 15s, postStep 30s, error 5s).
- Lesson 6: every nullable lifecycle field on the StandupSession row record is Optional<T>.
  The view table updater wraps values with Optional.of(...); callers use .orElse(...) or
  .isPresent().
- Lesson 7: StandupTasks.java with CONDUCT_STANDUP = Task.name(...).description(...)
  .resultConformsTo(StandupSummary.class) is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9181 declared explicitly in application.conf's
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
  Architecture, Risk Survey, Eval Matrix, App UI — no more. Any tab removed in an earlier
  iteration must be deleted from the HTML; display:none is not enough.
- The single-agent invariant: there is exactly ONE AutonomousAgent (ScrumMasterAgent). The
  ticket posting logic (TicketPostingService) is a plain service class — not an agent.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration
  mechanism, not as an external check that runs after the agent call returns. Lesson 1's
  AutonomousAgent contract is the authoritative reference.
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
