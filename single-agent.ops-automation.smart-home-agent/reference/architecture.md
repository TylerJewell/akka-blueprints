# Architecture — smart-home-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph centres on one AI decision — `HomeControlAgent` — that receives a natural-language command and the device catalogue, then returns a typed `DeviceAction`. `CommandEndpoint` accepts the resident's submission and writes a `CommandReceived` event onto `CommandEntity`. The same endpoint starts a `CommandWorkflow` instance. The workflow's `guardStep` calls `HomeControlAgent` — the single AutonomousAgent — with the raw command as `TaskDef.instructions(...)` and the device catalogue as a `TaskDef.attachment("devices.json", ...)`. The agent's before-tool-call guardrail (`ActionGuardrail`) validates each proposed action before it leaves the agent loop. Once an action passes, the workflow records `GuardPassed` on `CommandEntity`.

For sensitive actions (`LOCK`, `UNLOCK`, `SET_TEMPERATURE`), `confirmStep` emits `ConfirmationRequested` and pauses. The resident confirms or cancels via `CommandEndpoint`. On confirmation, `dispatchStep` calls `DeviceRegistry.applyAction(...)`, which updates the physical device state model and emits `DeviceStateChanged`. `CommandView` projects all `CommandEntity` events into a read-model row; `CommandEndpoint` serves this to the UI over REST and SSE.

The graph has no second agent. The HITL confirmation is a workflow step that interacts with `CommandEntity` commands — not an LLM call. That is what makes this a faithful **single-agent** example: exactly one component communicates with a model.

## Interaction sequence

The sequence traces the lock-with-HITL path (J2). Two distinct pauses occur:

1. The `guardStep` LLM call — bounded by the 60 s step timeout.
2. The `confirmStep` wait — the workflow is suspended until the resident POSTs confirm or cancel, or until the 300 s timeout expires.

Non-sensitive commands (light on/off, brightness) skip `confirmStep` entirely; `guardStep` transitions directly to `dispatchStep`.

## State machine

Seven reachable states. Key paths:

- Non-sensitive happy path: `RECEIVED → GUARD_PASSED → DISPATCHED`.
- Sensitive happy path: `RECEIVED → GUARD_PASSED → AWAITING_CONFIRMATION → DISPATCHED`.
- Resident cancels: `AWAITING_CONFIRMATION → CANCELLED`.
- Guardrail blocks: `RECEIVED → BLOCKED` (terminal).
- Agent error or dispatch failure: any non-terminal state → `FAILED`.

`BLOCKED`, `CANCELLED`, `DISPATCHED`, and `FAILED` are all terminal. A `BLOCKED` command's record is preserved on the entity — the UI shows the rejection reason so the resident can reformulate.

## Entity model

`CommandEntity` is the source of truth for the command lifecycle. `DeviceRegistry` is the source of truth for device state. `CommandWorkflow` reads from `DeviceRegistry` (to build the catalogue attachment) and writes to both `CommandEntity` (lifecycle events) and `DeviceRegistry` (via `applyAction`). `CommandView` projects `CommandEntity` events into the read model for the UI. The relationship between `HomeControlAgent` and `DeviceAction` is "returns" — the agent's task result is the action record that flows into the workflow.

## Defence-in-depth governance flow

For any action that reaches `DeviceRegistry.applyAction`, the command passed through:

1. **ActionGuardrail** — device-not-found, action-type-mismatch, and parameter-out-of-range are caught before the agent's response leaves the loop.
2. **HomeControlAgent** — one model call, one typed output.
3. **HITL confirmation** — for lock, unlock, and thermostat-override actions, an explicit resident approval is required before dispatch.

Each layer is independent. Removing the guardrail allows invalid device IDs and out-of-range parameters to reach the actuator layer. Removing the HITL step dispatches sensitive actions without resident awareness. Neither omission is silently covered by the other.
