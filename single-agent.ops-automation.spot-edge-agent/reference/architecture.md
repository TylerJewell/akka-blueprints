# Architecture — spot-edge-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `MissionEndpoint` accepts a mission submission, writes a `MissionSubmitted` event onto `MissionEntity`, and starts a `MissionWorkflow` instance. The workflow's `planStep` calls `SpotNavigatorAgent` — the single AutonomousAgent — with the mission brief as `TaskDef.instructions(...)` and the waypoint map as a `TaskDef.attachment(...)`. Every proposed Spot SDK command passes through `CommandGuardrail` before reaching the in-process `SimulatedSpotMcp`. If any command in the planning pass is classified as non-trivial, `approvalGateStep` pauses the workflow until an operator decision arrives via `MissionEndpoint.approve` or `.reject`. Once approved (or bypassed for trivial missions), `executeStep` runs the full mission through the agent again with live tool dispatch. Concurrently, `TelemetryMonitor` polls the simulated sensor bus; a threshold breach fires `SafetyHaltTriggered` directly onto `MissionEntity` and dispatches `SimulatedSpotMcp.halt()`, bypassing the agent entirely. `MissionView` projects every entity event into a read-model row; `MissionEndpoint` serves the read model over REST and SSE.

The graph has exactly one agent. `TelemetryMonitor` is a rule-based Consumer — it checks numeric thresholds, not intent — so the single-agent invariant is preserved: only `SpotNavigatorAgent` makes an LLM call.

## Interaction sequence

The sequence traces J1 (happy path, trivial 2-waypoint patrol). Two distinct moments where execution gates on something:

1. The `planStep` LLM call bounds the initial planning pass (30 s timeout). On a trivial mission this produces a COMPLETED outcome directly without entering the approval gate.
2. The `executeStep` LLM call runs the full dispatch sequence (120 s timeout). Each `spot.navigate_to` call waits on `SimulatedSpotMcp` before the agent proceeds to the next tool.

The telemetry polling runs in parallel throughout execution. It does not block the agent; it fires only when thresholds are crossed.

## State machine

Seven states. Notable paths:

- The happy path is `SUBMITTED → PLANNING → EXECUTING → COMPLETED`.
- The approval path is `SUBMITTED → PLANNING → PENDING_APPROVAL → EXECUTING → COMPLETED` (or `→ FAILED` on rejection or timeout).
- Two paths land in `HALTED`: a sensor threshold breach during `EXECUTING`. A halted mission retains all entity data — the operator sees the halt reason, sensor snapshot, and last command in the UI.
- Three paths land in `FAILED`: plan error, approval rejection, approval timeout.
- There is no `ROLLED_BACK` state. The system does not attempt to undo motion already executed; the operator decides the recovery action outside the system.

## Entity model

`MissionEntity` is the source of truth. It emits nine event types. `MissionView` projects all events into a read-model row used by the UI. `TelemetryMonitor` subscribes to entity events to detect execution start and then fires halt events independently. `MissionWorkflow` both reads (`getMission`) and writes (`startPlanning`, `requestApproval`, `decideApproval`, `startExecution`, `completeMission`, `recordFinalTelemetry`, `fail`) on the entity. The relationship between `SpotNavigatorAgent` and `MissionOutcome` is "returns" — the agent's task result is the outcome record.

## Defence-in-depth governance flow

For any command that reaches the robot, the mission passed through:

1. **CommandGuardrail** (`before-tool-call`) — every proposed Spot command is checked against forbidden zones, velocity caps, and terrain-gait compatibility before it touches the MCP channel. A bad command forces a replan, not an execution.
2. **Human approval gate** (`approvalGateStep`) — any non-trivial maneuver waits for an operator decision. The robot does not move until the decision arrives.
3. **TelemetryMonitor** (automatic safety halt) — live sensor readings feed an independent checker that fires outside the agent loop. Low battery, joint overload, obstacle proximity, and E-stop all trigger an immediate halt regardless of what the agent is doing.

Each layer is independent. A guardrail bypass (e.g., via a crafted instruction) still faces the approval gate for non-trivial commands and the safety halt for sensor anomalies.
