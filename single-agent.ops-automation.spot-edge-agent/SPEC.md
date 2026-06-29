# SPEC — spot-edge-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Spot Agent (Edge).
**One-line pitch:** An operator submits a high-level robot mission (inspect zone, navigate to waypoint, capture telemetry); one AI agent translates the intent into a sequence of Spot SDK motion commands, each validated by a `before-tool-call` guardrail before touching the robot, with a human-approval gate for non-trivial maneuvers and an automatic safety halt wired to live sensor feeds.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `SpotNavigatorAgent` (AutonomousAgent) carries the entire motion-planning decision; the surrounding components prepare its mission context and enforce three layers of physical safety.

- A **before-tool-call guardrail** intercepts every Spot SDK command before it is dispatched to the MCP server. Commands that target forbidden zones, exceed velocity limits, or request gaits incompatible with the known terrain map are rejected. The agent receives a structured rejection and must replan within its iteration budget.
- An **application-layer human-in-the-loop gate** pauses the workflow before the agent executes any maneuver classified as non-trivial (tipping, stair-climbing, payload manipulation). A human operator views a plain-English summary of the proposed command sequence and approves or rejects it. The gate is implemented as a workflow step with an explicit resume path.
- An **automatic safety halt** fires whenever `TelemetryMonitor` detects a sensor threshold breach — low battery, joint overload, E-stop signal, obstacle proximity violation. The halt bypasses the agent loop and writes `SafetyHaltTriggered` directly to `MissionEntity`, transitioning the mission to HALTED. Physical actuators are de-energized via the MCP server's `spot.halt` tool in the same step.

## 3. User-facing flows

The user opens the App UI tab.

1. The operator fills in the **Mission brief** form: mission name, a target zone or waypoint list (picked from the seeded waypoint map or typed as a custom route), payload instructions (e.g., "capture thermal image at each waypoint"), and the operator's callsign.
2. The operator clicks **Submit mission**. The UI POSTs to `/api/missions` and receives a `missionId`.
3. The mission card appears in the live list in `SUBMITTED` state. Within ~1 s, the workflow starts and the mission transitions to `PLANNING`.
4. If the mission's maneuver set is non-trivial, the card transitions to `PENDING_APPROVAL`. The right panel shows a plain-English command sequence summary and **Approve** / **Reject** buttons. The workflow pauses until the operator acts.
5. After approval (or immediately for trivial missions), the card transitions to `EXECUTING`. The agent begins issuing Spot SDK commands. The right panel streams live waypoint progress and telemetry tiles (battery %, joint load, GPS position).
6. On successful completion, the mission transitions to `COMPLETED`. A telemetry summary (distance covered, waypoints hit, anomalies flagged) appears on the card.
7. If a sensor threshold is breached at any point during execution, the mission transitions immediately to `HALTED`. The card displays the halt reason, the sensor snapshot at breach time, and the last agent-issued command.
8. The operator can submit a follow-up mission; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `MissionEndpoint` | `HttpEndpoint` | `/api/missions/*` — submit, list, get, approve, reject, SSE; serves `/api/metadata/*`. | — | `MissionEntity`, `MissionView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `MissionEntity` | `EventSourcedEntity` | Per-mission lifecycle: submitted → planning → pending approval → executing → completed/halted. Source of truth. | `MissionEndpoint`, `TelemetryMonitor`, `MissionWorkflow` | `MissionView` |
| `TelemetryMonitor` | `Consumer` | Subscribes to `MissionStarted` events; streams simulated sensor readings; emits `SafetyHaltTriggered` on threshold breach. | `MissionEntity` events | `MissionEntity` |
| `MissionWorkflow` | `Workflow` | One workflow per mission. Steps: `planStep` → `approvalGateStep` (conditional) → `executeStep` → `telemetryStep`. | started by `MissionEndpoint` after submit | `SpotNavigatorAgent`, `MissionEntity` |
| `SpotNavigatorAgent` | `AutonomousAgent` | The one decision-making LLM. Receives mission brief + waypoint map as task attachment; calls Spot SDK tools (simulated MCP); returns `MissionOutcome`. | invoked by `MissionWorkflow` | returns outcome |
| `CommandGuardrail` | supporting class | `before-tool-call` hook on `SpotNavigatorAgent`. Validates every proposed Spot command against terrain map, velocity limits, and forbidden-zone polygons. | wired to `SpotNavigatorAgent` | agent loop retry |
| `MissionView` | `View` | Read model: one row per mission for the UI. | `MissionEntity` events | `MissionEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
record Waypoint(String waypointId, String label, double lat, double lng, String terrainType) {}

record MissionBrief(
    String missionId,
    String missionName,
    List<Waypoint> route,
    String payloadInstructions,
    String operatorCallsign,
    boolean requiresApproval,
    Instant submittedAt
) {}

record CommandProposal(
    String commandId,
    String spotCommand,   // e.g. "spot.navigate_to", "spot.capture_image"
    Map<String, Object> params,
    String rationale
) {}

record ApprovalSummary(
    List<CommandProposal> proposedCommands,
    String plainEnglishSummary,
    ManeuverClass maneuverClass
) {}
enum ManeuverClass { TRIVIAL, NON_TRIVIAL }

record TelemetrySnapshot(
    double batteryPct,
    double jointLoadMax,   // 0.0–1.0
    double obstacleProximityM,
    boolean estopSignal,
    Instant capturedAt
) {}

record MissionOutcome(
    OutcomeStatus status,
    List<String> waypointsReached,
    List<String> anomaliesObserved,
    double distanceTravelledM,
    Instant completedAt
) {}
enum OutcomeStatus { COMPLETED, PARTIAL, ABORTED }

record HaltReason(
    String cause,          // e.g. "low-battery", "obstacle-proximity", "estop"
    TelemetrySnapshot snapshot,
    String lastCommand,
    Instant haltedAt
) {}

record Mission(
    String missionId,
    Optional<MissionBrief> brief,
    Optional<ApprovalSummary> approvalSummary,
    Optional<String> approvalDecision,   // "APPROVED" | "REJECTED"
    Optional<TelemetrySnapshot> lastTelemetry,
    Optional<MissionOutcome> outcome,
    Optional<HaltReason> haltReason,
    MissionStatus status,
    Instant createdAt,
    Optional<Instant> finishedAt
) {}

enum MissionStatus {
    SUBMITTED, PLANNING, PENDING_APPROVAL, EXECUTING, COMPLETED, HALTED, FAILED
}
```

Events on `MissionEntity`: `MissionSubmitted`, `PlanningStarted`, `ApprovalRequested`, `ApprovalDecided`, `MissionStarted`, `WaypointReached`, `SafetyHaltTriggered`, `MissionCompleted`, `MissionFailed`.

Every nullable lifecycle field on the `Mission` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/missions` — body `{ missionName, route: [Waypoint], payloadInstructions, operatorCallsign }` → `{ missionId }`.
- `GET /api/missions` — list all missions, newest-first.
- `GET /api/missions/{id}` — one mission.
- `POST /api/missions/{id}/approve` — operator approves a PENDING_APPROVAL mission; resumes workflow.
- `POST /api/missions/{id}/reject` — operator rejects a PENDING_APPROVAL mission; transitions to FAILED.
- `GET /api/missions/sse` — Server-Sent Events; one event per state transition.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Spot Agent (Edge)</title>`.

The App UI tab is a two-column layout: a left rail with the live list of submitted missions (status pill + outcome badge + age) and a right pane with the selected mission's detail — route waypoints, real-time telemetry tiles (battery %, joint load, proximity), approval summary (when pending), mission outcome, and halt reason (when halted).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: `CommandGuardrail` is registered on `SpotNavigatorAgent` via the agent's guardrail-configuration block, bound to the `before-tool-call` hook. On every proposed Spot command it checks (1) the target waypoint is not inside a forbidden-zone polygon, (2) the requested velocity does not exceed the terrain-appropriate limit, (3) the requested gait is compatible with the current terrain type, and (4) the command is in the allowed Spot SDK command set. On any failure, returns a structured rejection naming the violated constraint; the agent loop consumes one iteration and must replan. Passing commands reach the simulated MCP server.
- **H1 — automatic safety halt**: `TelemetryMonitor` Consumer subscribes to `MissionStarted` events and polls the simulated sensor bus every 500 ms during execution. On any threshold breach — battery below 15 %, joint load above 0.9, obstacle within 0.3 m, or E-stop signal asserted — the monitor calls `MissionEntity.triggerSafetyHalt(haltReason)` and dispatches `spot.halt` via the MCP tool. This path is entirely independent of the agent loop; the halt fires even if the agent is mid-iteration.
- **A1 — application-layer human-in-the-loop gate**: `MissionWorkflow.approvalGateStep` pauses execution whenever the mission's `requiresApproval` flag is true (set automatically when `SpotNavigatorAgent`'s planning pass classifies any command as `NON_TRIVIAL`). The step writes `ApprovalRequested` with a plain-English `ApprovalSummary` to the entity and suspends via `Workflow.pause()`. `MissionEndpoint` exposes `POST /api/missions/{id}/approve` and `POST /api/missions/{id}/reject`; those calls resume the workflow or fail the mission. The gate has a 10-minute timeout; an unanswered gate times out as a rejection.

## 9. Agent prompts

- `SpotNavigatorAgent` → `prompts/spot-navigator.md`. The single decision-making LLM. System prompt instructs it to plan a safe traversal of the supplied route, call Spot SDK tools in order, and return a structured `MissionOutcome`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Operator submits a 3-waypoint patrol mission (trivial maneuvers only); the workflow skips the approval gate; the agent executes all waypoints; `MissionOutcome.status = COMPLETED` appears in the UI within 30 s.
2. **J2** — Agent proposes a stair-climb command on a flat-terrain mission; `CommandGuardrail` rejects it (terrain mismatch); the agent replans via a flat corridor; the mission completes without a stair-climb command in the log.
3. **J3** — A simulated battery reading drops to 10 % at waypoint 2 of 4; `TelemetryMonitor` fires the safety halt; the mission transitions to HALTED; the UI shows the battery snapshot and the last command executed.
4. **J4** — A payload-manipulation mission triggers `requiresApproval = true`; the UI shows the approval panel; the operator rejects; the mission transitions to FAILED with `approvalDecision = REJECTED` visible on the card.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named spot-edge-agent demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services; Spot MCP server is simulated in-process).
Maven group io.akka.samples. Maven artifact single-agent-ops-automation-spot-edge-agent.
Java package io.akka.samples.spotagentedge. Akka 3.6.0. HTTP port 9510.

Components to wire (exactly):

- 1 AutonomousAgent SpotNavigatorAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/spot-navigator.md>) and
  .capability(TaskAcceptance.of(EXECUTE_MISSION).maxIterationsPerTask(5)). The task receives
  the mission brief as its instruction text and the waypoint map as a task ATTACHMENT
  (NOT as inline prompt text — TaskDef.attachment("waypoints.json", waypointsBytes) is the
  canonical call). The agent is configured with a before-tool-call guardrail (see G1 in
  eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 5-iteration budget. Output: MissionOutcome
  {status: OutcomeStatus, waypointsReached: List<String>, anomaliesObserved: List<String>,
  distanceTravelledM: double, completedAt: Instant}.

- 1 Workflow MissionWorkflow per missionId with four steps:
  * planStep — calls SpotNavigatorAgent.runSingleTask with EXECUTE_MISSION task to perform
    a dry-run planning pass (agent returns a CommandProposal list plus ManeuverClass). If
    any command is NON_TRIVIAL sets requiresApproval = true, writes ApprovalRequested, then
    transitions to approvalGateStep; otherwise transitions directly to executeStep.
    WorkflowSettings.stepTimeout 30s.
  * approvalGateStep — pauses via Workflow.pause(); resumes on MissionEndpoint.approve or
    .reject. On reject transitions to error. Timeout 10 minutes; timeout fires reject.
    WorkflowSettings.stepTimeout 600s.
  * executeStep — emits MissionStarted, then calls SpotNavigatorAgent.runSingleTask with
    EXECUTE_MISSION and the full waypoint attachment. Dispatches each CommandProposal through
    the simulated MCP (SimulatedSpotMcp.execute(commandId, params)). On success calls
    MissionEntity.completeMission(outcome). WorkflowSettings.stepTimeout 120s with
    defaultStepRecovery maxRetries(2).failoverTo(MissionWorkflow::error).
  * telemetryStep — after executeStep succeeds, waits 2s then calls
    MissionEntity.recordFinalTelemetry. WorkflowSettings.stepTimeout 10s.
  error step transitions the entity to FAILED.

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity MissionEntity (one per missionId). State Mission{missionId: String,
  brief: Optional<MissionBrief>, approvalSummary: Optional<ApprovalSummary>,
  approvalDecision: Optional<String>, lastTelemetry: Optional<TelemetrySnapshot>,
  outcome: Optional<MissionOutcome>, haltReason: Optional<HaltReason>,
  status: MissionStatus, createdAt: Instant, finishedAt: Optional<Instant>}.
  MissionStatus enum: SUBMITTED, PLANNING, PENDING_APPROVAL, EXECUTING, COMPLETED, HALTED,
  FAILED. Events: MissionSubmitted{brief}, PlanningStarted{}, ApprovalRequested{summary},
  ApprovalDecided{decision}, MissionStarted{}, WaypointReached{waypointId},
  SafetyHaltTriggered{haltReason}, MissionCompleted{outcome}, MissionFailed{reason}.
  Commands: submit, startPlanning, requestApproval, decideApproval, startExecution,
  recordWaypoint, triggerSafetyHalt, completeMission, recordFinalTelemetry, fail,
  getMission. emptyState() returns Mission.initial("") with no commandContext() reference
  (Lesson 3). Every Optional<T> field uses Optional.empty() in initial state and
  Optional.of(...) inside the event-applier.

- 1 Consumer TelemetryMonitor subscribed to MissionEntity events; on MissionStarted starts
  polling SimulatedSpotMcp.readTelemetry() every 500 ms until MissionCompleted or
  MissionFailed or SafetyHaltTriggered arrives. On any threshold breach (battery < 15%,
  jointLoadMax > 0.9, obstacleProximityM < 0.3, estopSignal == true) calls
  MissionEntity.triggerSafetyHalt(haltReason) and dispatches SimulatedSpotMcp.halt().
  The Consumer does NOT call the agent — it acts independently of the agent loop.

- 1 View MissionView with row type MissionRow (mirrors Mission minus verbose telemetry
  history — the view keeps only lastTelemetry for the live tiles). Table updater consumes
  MissionEntity events. ONE query getAllMissions: SELECT * AS missions FROM mission_view.
  No WHERE status filter (Lesson 2); caller filters client-side.

- 2 HttpEndpoints:
  * MissionEndpoint at /api with POST /missions (body {missionName, route: [{waypointId,
    label, lat, lng, terrainType}], payloadInstructions, operatorCallsign}; mints missionId;
    calls MissionEntity.submit; starts MissionWorkflow; returns {missionId}), GET /missions
    (list from getAllMissions, sorted newest-first), GET /missions/{id} (one row), POST
    /missions/{id}/approve (calls MissionEntity.decideApproval("APPROVED"); resumes
    MissionWorkflow), POST /missions/{id}/reject (calls MissionEntity.decideApproval(
    "REJECTED")), GET /missions/sse (Server-Sent Events forwarded from the view's
    stream-updates), and three /api/metadata/* endpoints serving the YAML/MD files.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- MissionTasks.java declaring one Task<R> constant: EXECUTE_MISSION = Task.name("Execute
  mission").description("Plan and execute a Spot robot mission from the attached waypoint
  map; return a MissionOutcome").resultConformsTo(MissionOutcome.class). DO NOT skip this
  — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records Waypoint, MissionBrief, CommandProposal, ApprovalSummary, ManeuverClass,
  TelemetrySnapshot, MissionOutcome, OutcomeStatus, HaltReason, Mission, MissionStatus.

- CommandGuardrail.java implementing the before-tool-call hook. Reads the proposed Spot
  command and params, runs the four checks listed in eval-matrix.yaml G1, and either
  passes through or returns Guardrail.reject(<structured-error>) to force the agent to
  replan.

- SimulatedSpotMcp.java — an in-process simulation of the Spot MCP server. Implements
  the Spot tool calls: spot.navigate_to, spot.capture_image, spot.change_gait,
  spot.dock, spot.halt, spot.read_telemetry. Returns deterministic simulated responses
  seeded from waypoints.jsonl. The simulation injects a battery-low event after 6 tool
  calls by default (configurable via application.conf).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9510, the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading ${?ANTHROPIC_API_KEY}, ${?OPENAI_API_KEY},
  ${?GOOGLE_AI_GEMINI_API_KEY}, and simulation.battery-inject-after-calls = 6.

- src/main/resources/sample-events/waypoints.jsonl with 3 seeded route configurations:
  a 3-waypoint indoor patrol (flat terrain, trivial), a 5-waypoint outdoor inspection
  (mixed terrain, includes one stair segment — non-trivial), and a 2-waypoint payload-drop
  mission (tipping maneuver — non-trivial, approval required).

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md (copies of
  the root-level files for the endpoint to serve from classpath).

- eval-matrix.yaml at the project root with 3 controls (G1, H1, A1) matching the
  mechanisms in Section 8 of this SPEC. Matching simplified_view list.

- risk-survey.yaml at the project root with sector = industrial-robotics,
  data.data_classes.pii = false, decisions.authority_level = autonomous-with-oversight
  (the agent executes physical commands; human approves non-trivial ones),
  oversight.human_in_loop = true (approval gate), oversight.human_on_loop = true
  (operator monitors live telemetry), failure.failure_modes including
  "physical-collision", "joint-overload", "forbidden-zone-entry", "stale-waypoint-map",
  "guardrail-bypass-via-prompt-injection"; deployer fields marked
  TO_BE_COMPLETED_BY_DEPLOYER.

- prompts/spot-navigator.md loaded as the agent system prompt.

- README.md at the project root: title "Akka Sample: Spot Agent (Edge)", prerequisites,
  generate-the-system, what-you-get, customise-before-generating, what-gets-validated,
  license. NO Configuration section. NO governance-mechanisms section (Lesson 20).

- src/main/resources/static-resources/index.html — single self-contained file (no ui/,
  no npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout
  (left = live list of mission cards; right = selected-mission detail with route
  waypoints, real-time telemetry tiles, approval summary panel, outcome, and halt reason).
  Browser title exactly: <title>Akka Sample: Spot Agent (Edge)</title>. No subtitle on
  the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-
        shape-correct outputs per agent (see Mock LLM provider block below).
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.
- The generated Bootstrap.java fails fast with a clear message naming the configured key
  reference if it does not resolve at runtime.

Mock LLM provider — required when option (a) is selected:

- Generate MockModelProvider.java with per-task dispatch on Task<R> id. For EXECUTE_MISSION:
  read src/main/resources/mock-responses/execute-mission.json, pick one entry
  pseudo-randomly per call seeded from missionId.
- execute-mission.json — 6 MissionOutcome entries covering COMPLETED and PARTIAL statuses.
  Each entry has a waypointsReached list, an anomaliesObserved list (some empty, some with
  "unexpected-obstacle", "low-light-detected"), a realistic distanceTravelledM, and a
  completedAt. Plus 2 entries with a deliberate CommandProposal whose spotCommand is not
  in the allowed set — these trigger the CommandGuardrail, exercising the replan path.
  The mock selects a guardrail-triggering entry on the FIRST iteration of every 4th mission
  (modulo seed) so J2 is reproducible.
- A MockModelProvider.seedFor(missionId) helper makes per-mission selection deterministic
  across restarts.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md
for the full list. Notably:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. SpotNavigatorAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion MissionTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (planStep 30s, approvalGateStep
  600s, executeStep 120s, telemetryStep 10s, error 10s).
- Lesson 6: every nullable lifecycle field on Mission is Optional<T>.
- Lesson 7: MissionTasks.java with EXECUTE_MISSION is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash. No deprecated gemini-2.0-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9510 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box" — never T1/T2/T3/T4.
- Lesson 23: no forbidden words.
- Lesson 24: the generated static-resources/index.html includes mermaid CSS overrides
  and themeVariables block.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching matches by data-tab / data-panel attribute, NEVER by NodeList
  index. Exactly five <section class="tab-panel"> elements.
- The single-agent invariant: exactly ONE AutonomousAgent (SpotNavigatorAgent). The
  TelemetryMonitor Consumer is NOT an agent — it is a rule-based threshold checker.
- The waypoint map is passed as a Task ATTACHMENT, not inlined into the instruction text.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration block,
  not as an external check after the command is dispatched. It must intercept the proposed
  tool call before any MCP channel activity.
- The human-approval gate is implemented as Workflow.pause() inside approvalGateStep, with
  a resume path wired to MissionEndpoint. It is NOT a polling loop.
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
