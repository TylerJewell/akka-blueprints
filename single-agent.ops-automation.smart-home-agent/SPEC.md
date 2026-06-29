# SPEC — smart-home-agent

The natural-language brief `/akka:specify` reads to generate this system.

---

## 1. System name + pitch

**System name:** Connected House Agent.
**One-line pitch:** A resident types a natural-language command ("lock the front door", "set the thermostat to 70", "turn on the kitchen lights"); one AI agent resolves the intent, selects the target devices, and emits a typed `DeviceAction` — with a before-tool-call guardrail blocking unauthorized or out-of-range commands and a HITL confirmation step gating any physically consequential action.

## 2. What this blueprint demonstrates

The **single-agent** coordination pattern in the ops-automation domain. One `HomeControlAgent` (AutonomousAgent) carries the entire intent-resolution and action-selection decision; the surrounding components only validate its proposed actions and record their outcomes. Two governance mechanisms are wired around the agent:

- A **before-tool-call guardrail** runs each time the agent proposes a device action. It checks that the target device exists in the registry, that the action type is permitted for that device class, and that any numeric parameter (temperature set-point, brightness level) falls within the device's allowed range. Commands that fail are returned to the agent as structured errors so it can reformulate; the guardrail never throws silently.
- A **HITL application confirmation** gates lock/unlock and thermostat-override commands. The workflow pauses, presents the proposed action to the resident, and only dispatches once the resident confirms. Rejection cancels the action cleanly.

The blueprint shows that single-agent does not mean ungoverned — two independent checks sit between intent and actuation.

## 3. User-facing flows

The user opens the App UI tab.

1. The resident types a command into the **Command** text input (or picks one of four seeded examples — "Turn on the kitchen lights", "Lock the front door", "Set the thermostat to 68°F", "Dim the living room lights to 40%").
2. The resident clicks **Send command**. The UI POSTs to `/api/commands` and receives a `commandId`.
3. The card appears in the live list with status `RECEIVED`. Within ~1 s the workflow's `guardStep` runs: if the guardrail passes, the card advances to `GUARD_PASSED`; if the guardrail rejects, the card transitions to `BLOCKED` and the rejection reason is visible on the card.
4. For non-sensitive commands (light on/off, brightness), the workflow dispatches immediately. The card reaches `DISPATCHED` within ~10–30 s (LLM call). The dispatched action is shown: device name, action type, parameter if any.
5. For sensitive commands (lock/unlock, thermostat override), the card reaches `AWAITING_CONFIRMATION`. A confirmation banner appears on the card with the proposed action and two buttons: **Confirm** and **Cancel**. The resident clicks one. On **Confirm** the card transitions to `DISPATCHED`; on **Cancel** it transitions to `CANCELLED`.
6. The resident can submit additional commands; the live list keeps history visible.

## 4. Components

| Component | Akka primitive | Role | Upstream | Downstream |
|---|---|---|---|---|
| `CommandEndpoint` | `HttpEndpoint` | `/api/commands/*` — submit, list, get, SSE, confirm/cancel; serves `/api/metadata/*`. | — | `CommandEntity`, `CommandView` |
| `AppEndpoint` | `HttpEndpoint` | Serves `/` → `/app/index.html` and `/app/*`. | — | static resources |
| `CommandEntity` | `EventSourcedEntity` | Per-command lifecycle: received → guard_passed/blocked → awaiting_confirmation/dispatched/cancelled. Source of truth. | `CommandEndpoint`, `CommandWorkflow` | `CommandView` |
| `DeviceRegistry` | `EventSourcedEntity` | Holds the current state of every device (on/off, brightness 0–100, temperature set-point, lock status). One entity per home (keyed by `homeId`). | `CommandEndpoint` (register/update device), `CommandWorkflow` (read state) | `CommandView` |
| `CommandWorkflow` | `Workflow` | One workflow per command. Steps: `guardStep` → `confirmStep` (sensitive only) → `dispatchStep`. | started by `CommandEntity` after `CommandReceived` | `HomeControlAgent`, `CommandEntity`, `DeviceRegistry` |
| `HomeControlAgent` | `AutonomousAgent` | The single decision-making LLM. Receives the resident's command plus the current device catalogue as task context; returns a `DeviceAction`. | invoked by `CommandWorkflow` | returns DeviceAction |
| `ActionGuardrail` | (supporting class) | before-tool-call hook registered on `HomeControlAgent`. Validates target device, action type, and numeric range. | agent loop | returns accept or structured rejection |
| `CommandView` | `View` | Read model: one row per command for the UI. | `CommandEntity` events | `CommandEndpoint` |

Names matter — `/akka:specify` will use them verbatim.

## 5. Data model

```java
enum DeviceClass { LIGHT, THERMOSTAT, LOCK, BLIND }
enum ActionType  { TURN_ON, TURN_OFF, SET_BRIGHTNESS, SET_TEMPERATURE, LOCK, UNLOCK, OPEN, CLOSE }
enum CommandStatus {
    RECEIVED, GUARD_PASSED, BLOCKED, AWAITING_CONFIRMATION,
    DISPATCHED, CANCELLED, FAILED
}

record Device(
    String deviceId,
    String displayName,
    DeviceClass deviceClass,
    String location,            // e.g. "kitchen", "living room", "front door"
    int    minValue,            // lower bound for numeric params (e.g. 60 for thermostat °F)
    int    maxValue             // upper bound
) {}

record DeviceState(
    String  deviceId,
    boolean on,
    Integer brightnessPercent,  // LIGHT only, null otherwise
    Integer temperatureSetPoint,// THERMOSTAT only
    boolean locked              // LOCK only
) {}

record CommandRequest(
    String commandId,
    String homeId,
    String rawCommand,          // resident's natural-language input
    String submittedBy,
    Instant submittedAt
) {}

record DeviceAction(
    String     deviceId,
    ActionType actionType,
    Integer    numericParam,    // brightness %, temperature °F, or null
    String     rationale        // one sentence explaining the action choice
) {}

record GuardResult(
    boolean   passed,
    String    rejectionReason   // null when passed
) {}

record Command(
    String                  commandId,
    Optional<CommandRequest> request,
    Optional<DeviceAction>  action,
    Optional<GuardResult>   guardResult,
    CommandStatus           status,
    Instant                 createdAt,
    Optional<Instant>       finishedAt
) {}
```

Events on `CommandEntity`: `CommandReceived`, `GuardPassed`, `CommandBlocked`, `ConfirmationRequested`, `ActionConfirmed`, `ActionCancelled`, `ActionDispatched`, `CommandFailed`.

Every nullable lifecycle field on the `Command` row record is `Optional<T>` (Lesson 6).

See `reference/data-model.md`.

## 6. API contract

- `POST /api/commands` — body `{ homeId, rawCommand, submittedBy }` → `{ commandId }`.
- `GET /api/commands` — list all commands, newest-first.
- `GET /api/commands/{id}` — one command.
- `GET /api/commands/sse` — Server-Sent Events; one event per state transition.
- `POST /api/commands/{id}/confirm` — resident confirms a pending HITL action.
- `POST /api/commands/{id}/cancel` — resident cancels a pending HITL action.
- `GET /api/devices` — list all devices in the default home.
- `GET /api/metadata/{readme,risk-survey,eval-matrix}` — UI metadata.
- `GET /` → `/app/index.html`. `GET /app/*` — static UI.

See `reference/api-contract.md` for payload schemas and SSE event format.

## 7. UI

Five-tab structure inherited from the formal exemplar (see `reference/ui-mockup.md`). Browser title: `<title>Akka Sample: Connected House Agent</title>`.

The App UI tab is a two-column layout: a left rail showing a command input panel, the seeded device grid (showing current device state), and the live command list (status pill + action badge + age); a right pane showing the selected command's detail — raw command, guard result, proposed action, and the confirmation panel (for AWAITING_CONFIRMATION commands).

Tab switching must match by `data-tab` / `data-panel` attribute, never by NodeList index (Lesson 26). Mermaid diagrams in the Architecture tab must include the CSS overrides and `themeVariables` block from Lesson 24 — without them, state-diagram labels render black-on-black and edge labels clip.

## 8. Governance

See `eval-matrix.yaml` and `risk-survey.yaml`. The generated system wires:

- **G1 — before-tool-call guardrail**: runs every time `HomeControlAgent` proposes a `DeviceAction`. `ActionGuardrail` asserts (1) the `deviceId` is present in `DeviceRegistry`, (2) the `actionType` is valid for that device's `DeviceClass`, and (3) any `numericParam` is within `[device.minValue, device.maxValue]`. On failure, returns a structured `invalid-action` rejection naming the failed check; the agent loop retries within its iteration budget.
- **H1 — HITL application confirmation**: `CommandWorkflow.confirmStep` fires when `HomeControlAgent` returns an action with `actionType` in `{LOCK, UNLOCK, SET_TEMPERATURE}`. The workflow pauses (`WorkflowSettings.stepTimeout 300s`) waiting for `CommandEndpoint` to receive a `POST /api/commands/{id}/confirm` or `POST /api/commands/{id}/cancel` from the resident. Only on `confirm` does the workflow advance to `dispatchStep`.

## 9. Agent prompts

- `HomeControlAgent` → `prompts/home-control-agent.md`. The single decision-making LLM. System prompt instructs it to parse the resident's command, select the correct device from the catalogue, choose the appropriate `ActionType`, and return one `DeviceAction`.

## 10. Acceptance

See `reference/user-journeys.md`:

1. **J1** — Resident submits "Turn on the kitchen lights"; the command dispatches without confirmation within 30 s; the kitchen light device state flips to `on`.
2. **J2** — Resident submits "Lock the front door"; the HITL confirmation step fires; the resident confirms; the lock device state flips to `locked = true`.
3. **J3** — Resident submits "Set the thermostat to 95°F" (above the device's `maxValue` of 80); the guardrail blocks it with a structured rejection naming the range violation; the card reaches `BLOCKED` and never advances to confirmation or dispatch.
4. **J4** — Resident submits "Turn off the garage door" (a device class not in the registry); the guardrail blocks it with a `device-not-found` rejection; the card reaches `BLOCKED`.

---

## 11. Implementation directives

The Akka-specific details an implementing agent follows to produce the system. SPEC.md as a whole — Sections 1–11 — is the input to `/akka:specify @SPEC.md`.

```
Create a sample named smart-home-agent demonstrating the single-agent × ops-automation cell.
Runs out of the box (no external services). Maven group io.akka.samples. Maven artifact
single-agent-ops-automation-smart-home-agent. Java package
io.akka.samples.connectedhouseagent. Akka 3.6.0. HTTP port 9661.

Components to wire (exactly):

- 1 AutonomousAgent HomeControlAgent — definition() returns AgentDefinition with
  .instructions(<system prompt loaded from prompts/home-control-agent.md>) and
  .capability(TaskAcceptance.of(CONTROL_DEVICE).maxIterationsPerTask(3)). The task receives
  the resident's rawCommand as its instruction text, plus the current device catalogue (from
  DeviceRegistry) serialised as a task ATTACHMENT named "devices.json" (NOT as inline prompt
  text — Akka's TaskDef.attachment(name, contentBytes) is the canonical call). Output:
  DeviceAction{deviceId: String, actionType: ActionType, numericParam: Integer (nullable),
  rationale: String}. The agent is configured with a before-tool-call guardrail (see G1 in
  eval-matrix.yaml) registered via the agent's guardrail-configuration block. On guardrail
  rejection the agent loop retries within its 3-iteration budget.

- 1 Workflow CommandWorkflow per commandId with three steps:
  * guardStep — calls HomeControlAgent.runSingleTask(TaskDef.instructions(rawCommand)
    .attachment("devices.json", catalogueBytes)); the guardrail runs inside the agent loop.
    On GUARD_PASSED (guardrail accepted all proposed tool calls), calls
    CommandEntity.recordGuardPassed(action). WorkflowSettings.stepTimeout 60s with
    defaultStepRecovery maxRetries(2).failoverTo(CommandWorkflow::error).
  * confirmStep — runs only if action.actionType in {LOCK, UNLOCK, SET_TEMPERATURE}.
    Emits ConfirmationRequested on the entity. Pauses (WorkflowSettings.stepTimeout 300s)
    waiting for CommandEntity.confirm() or CommandEntity.cancel() to be called by the
    endpoint. On confirm advances to dispatchStep; on cancel emits ActionCancelled and ends.
    For non-sensitive actions, guardStep transitions directly to dispatchStep.
  * dispatchStep — calls DeviceRegistry.applyAction(deviceId, actionType, numericParam).
    Emits ActionDispatched on CommandEntity. WorkflowSettings.stepTimeout 10s. error step
    calls CommandEntity.fail(reason).

  WorkflowSettings is the nested Workflow.WorkflowSettings — DO NOT import a top-level
  WorkflowSettings class (Lesson 5).

- 1 EventSourcedEntity CommandEntity (one per commandId). State Command{commandId: String,
  request: Optional<CommandRequest>, action: Optional<DeviceAction>,
  guardResult: Optional<GuardResult>, status: CommandStatus, createdAt: Instant,
  finishedAt: Optional<Instant>}. CommandStatus enum: RECEIVED, GUARD_PASSED, BLOCKED,
  AWAITING_CONFIRMATION, DISPATCHED, CANCELLED, FAILED. Events: CommandReceived{request},
  GuardPassed{action, guardResult}, CommandBlocked{reason}, ConfirmationRequested{action},
  ActionConfirmed{}, ActionCancelled{}, ActionDispatched{action}, CommandFailed{reason}.
  Commands: submit, recordGuardPassed, blockCommand, requestConfirmation, confirm, cancel,
  dispatch, fail, getCommand. emptyState() returns Command.initial("") with no
  commandContext() reference (Lesson 3). Every Optional<T> field uses Optional.empty() in
  initial state and Optional.of(...) inside event-appliers.

- 1 EventSourcedEntity DeviceRegistry (one per homeId, default homeId = "home-1"). State
  holds Map<String, Device> catalogue and Map<String, DeviceState> currentState. Commands:
  registerDevice(Device), applyAction(deviceId, actionType, numericParam), getDevices,
  getDeviceState(deviceId). Events: DeviceRegistered{device}, DeviceStateChanged{deviceId,
  newState}. emptyState() returns empty maps; registers 4 seeded devices on first access
  (kitchen-light, living-room-light, front-door-lock, thermostat-main) via a Bootstrap
  component that calls registerDevice if the registry is empty.

- 1 View CommandView with row type CommandRow (mirrors Command). Table updater consumes
  CommandEntity events. ONE query getAllCommands: SELECT * AS commands FROM command_view.
  No WHERE status filter — Akka cannot auto-index enum columns (Lesson 2); caller filters
  client-side.

- 2 HttpEndpoints:
  * CommandEndpoint at /api with POST /commands (body {homeId, rawCommand, submittedBy};
    mints commandId; calls CommandEntity.submit; starts CommandWorkflow; returns {commandId}),
    GET /commands (list from getAllCommands, sorted newest-first), GET /commands/{id} (one
    row), GET /commands/sse (SSE from view stream-updates), POST /commands/{id}/confirm
    (calls CommandEntity.confirm), POST /commands/{id}/cancel (calls CommandEntity.cancel),
    GET /devices (returns DeviceRegistry.getDevices for homeId "home-1"), and three
    /api/metadata/* endpoints serving the YAML/MD files from
    src/main/resources/metadata/.
  * AppEndpoint serving GET / -> 302 /app/index.html and GET /app/* -> static-resources/*.

Companion files:

- CommandTasks.java declaring one Task<R> constant: CONTROL_DEVICE = Task.name("Control
  device").description("Parse the resident's command, select the target device from the
  catalogue, and return a DeviceAction").resultConformsTo(DeviceAction.class). DO NOT skip
  this — the AutonomousAgent requires its companion Tasks class (Lesson 7).

- Domain records: Device, DeviceState, DeviceClass (enum), ActionType (enum), CommandRequest,
  DeviceAction, GuardResult, Command, CommandStatus (enum).

- ActionGuardrail.java implementing the before-tool-call hook. Reads the candidate
  DeviceAction from the agent's proposed tool call, runs the three checks listed in
  eval-matrix.yaml G1, and either passes the call through or returns
  Guardrail.reject(<structured-error>) naming the failed check (device-not-found,
  action-type-mismatch, or parameter-out-of-range).

- src/main/resources/application.conf with akka.javasdk.dev-mode.http-port = 9661 and the
  three model-provider blocks (anthropic claude-sonnet-4-6, openai gpt-4o, googleai-gemini
  gemini-2.5-flash) reading the canonical env vars ${?ANTHROPIC_API_KEY},
  ${?OPENAI_API_KEY}, ${?GOOGLE_AI_GEMINI_API_KEY}.

- src/main/resources/sample-events/seed-commands.jsonl with 4 seeded commands covering the
  four device classes: a light on, a thermostat set, a lock, and a brightness dim.

- src/main/resources/metadata/eval-matrix.yaml, risk-survey.yaml, README.md.

- eval-matrix.yaml at the project root with 2 controls (G1, H1) matching Section 8.

- risk-survey.yaml at the project root with pre-filled ops-automation answers (see
  risk-survey.yaml).

- prompts/home-control-agent.md loaded as the agent system prompt.

- README.md at the project root.

- src/main/resources/static-resources/index.html — single self-contained file (no ui/, no
  npm). Five tabs matching the formal exemplar. App UI tab uses a two-column layout.
  Browser title exactly: <title>Akka Sample: Connected House Agent</title>. No subtitle
  on the Overview tab.

Generation workflow — see AKKA-EXEMPLAR-LESSONS.md Lesson 25:

- BEFORE scaffolding files, /akka:specify must inspect the environment for
  ANTHROPIC_API_KEY / OPENAI_API_KEY / GOOGLE_AI_GEMINI_API_KEY. If exactly one is set,
  default application.conf's model-provider to match and proceed silently.
- If none is set, ask the user how to source the key, offering five options:
    (a) Mock LLM — no real key; generate a MockModelProvider that returns random-but-shape-
        correct DeviceAction outputs per command. Sets model-provider = mock.
    (b) Name an existing env var.
    (c) Point to an existing env file.
    (d) Secrets-store URI.
    (e) Type once in this session.
- NEVER write the key value to any file Akka creates.

Constraints — see explainability/specs/eval-matrix/blueprint-program/AKKA-EXEMPLAR-LESSONS.md:

- Lesson 1: AutonomousAgent never silently downgraded to Agent. HomeControlAgent extends
  akka.javasdk.agent.autonomous.AutonomousAgent. The companion CommandTasks.java MUST exist.
- Lesson 4: every workflow step has an explicit stepTimeout (guardStep 60s, confirmStep 300s,
  dispatchStep 10s, error 5s).
- Lesson 6: every nullable lifecycle field on Command is Optional<T>.
- Lesson 7: CommandTasks.java with CONTROL_DEVICE is mandatory.
- Lesson 8: model names verified — anthropic claude-sonnet-4-6, openai gpt-4o,
  googleai-gemini gemini-2.5-flash.
- Lesson 9: run command is "/akka:build" — never "mvn akka:run".
- Lesson 10: port 9661 declared explicitly in application.conf.
- Lesson 11: no source-platform metadata appears anywhere user-facing.
- Lesson 12: UI fits 1080px content column with no horizontal scroll.
- Lesson 13: integration label "Runs out of the box".
- Lesson 23: no forbidden words in any user-facing prose.
- Lesson 24: static-resources/index.html includes mermaid CSS overrides and themeVariables.
- Lesson 25: API-key sourcing — NEVER write the key value to disk.
- Lesson 26: tab switching by data-tab / data-panel attribute; exactly five tab-panel sections.
- Single-agent invariant: exactly ONE AutonomousAgent (HomeControlAgent). The HITL
  confirmation is a workflow step calling CommandEntity commands — not a second LLM.
- The device catalogue is passed as a task ATTACHMENT ("devices.json"), never inlined.
- The before-tool-call guardrail is wired via the agent's guardrail-configuration block.
```

## 12. Post-scaffolding workflow — auto-chain the rest of SDD

When `/akka:specify` finishes scaffolding the SDD specification into `specs/features/`, **do not stop and wait for the user**. Immediately continue through the rest of the SDD pipeline without prompting:

1. Run `/akka:plan` — produce the architectural plan from the freshly-scaffolded spec. Accept defaults; pick the most conservative option if anything is ambiguous, consistent with Sections 4 (Components) and 6 (PLAN.md).
2. Run `/akka:tasks` — break the plan into implementation tasks.
3. Run `/akka:implement` — execute every task in sequence. If a task fails, capture the failure, continue with the next task, and surface a summary of all failures at the end.
4. Run `/akka:build` — compile and start the service via the Akka MCP tools.

Once `/akka:build` reports the service is running, output **just** the listening URL and a one-line summary of any failures from step 3. Do not narrate intermediate progress unless a step takes longer than 30 seconds without output.

**Stop earlier only** if a step returns a hard error you cannot work around without asking the user — missing API key, an unrecoverable compile error after exhausting auto-fix options, or a network failure on a required download.

## 13. Lessons applied

This blueprint follows the full Akka exemplar-lessons set. The constraints in Section 11 explicitly call out the lessons most load-bearing for this pattern; the remainder are honored by the authoring template. Lessons applied: Lesson 1, Lesson 2, Lesson 3, Lesson 4, Lesson 5, Lesson 6, Lesson 7, Lesson 8, Lesson 9, Lesson 10, Lesson 11, Lesson 12, Lesson 13, Lesson 14, Lesson 15, Lesson 16, Lesson 17, Lesson 18, Lesson 19, Lesson 20, Lesson 21, Lesson 22, Lesson 23, Lesson 24, Lesson 25, Lesson 26.
