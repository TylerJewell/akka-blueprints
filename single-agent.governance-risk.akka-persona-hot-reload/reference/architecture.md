# Architecture — persona-hot-reload

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `PersonaEndpoint` accepts a persona push and writes a `PersonaChangeProposed` event onto `PersonaEntity`. `PersonaChangeConsumer` subscribes to that event, passes the payload to `PersonaValidator`, and either activates or rejects the change. If the change is valid, the Consumer starts a `PersonaWatchWorkflow` instance. The workflow's `activateAgentStep` calls `PersonaAgent.rebuildDefinition(snapshot)` — the single AutonomousAgent updates its definition in-place. `revalidationStep` then sends a fixed probe set through `PersonaAgent` and passes the responses to `BehavioralRevalidator` for scoring. `monitoringStep` opens a 300-second observation window during which `PersonaChangeConsumer` forwards live agent responses to the configured operator webhook. `PersonaView` projects every entity event into a read-model row; `PersonaEndpoint` serves the read model and the chat interface over REST and SSE.

The graph deliberately has no second agent. `BehavioralRevalidator` is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct moments where the system waits:

1. The `PersonaValidator` check inside `PersonaChangeConsumer` — synchronous, sub-millisecond in normal operation; causes a `PersonaRejected` event if the check fails before any workflow starts.
2. The `awaitValidationStep` polling loop in the workflow — polls `PersonaEntity` every 1 s up to its 10 s timeout, advancing when `change.status == ACTIVATING`.
3. The `revalidationStep` probe loop — sends N probe queries to `PersonaAgent` back-to-back, bounded by the 60 s step timeout. Each probe is an independent LLM call.

The `monitoringStep` runs asynchronously for 300 seconds. The LLM call for any individual user chat query is unbounded by a step timeout (it runs outside the workflow); the workflow only issues probe calls during `revalidationStep`.

## State machine

Seven states. The interesting paths:

- The happy path is `VALIDATING → ACTIVATING → ACTIVE → REVALIDATING → MONITORED`.
- `REJECTED` is a terminal path for payloads that fail the gate check. The persona stays on the previous version; the entity stores the rejection reason.
- `FAILED` is reached from `ACTIVATING` (agent rebuild error) or from `REVALIDATING` (workflow error, not a behavioral probe failure — a FAIL probe result still transitions to `MONITORED`).
- There is no automatic rollback on a FAIL revalidation badge. The operator reads the result and pushes a corrective persona change. The blueprint stops at `MONITORED`; downstream enforcement is the deployer's responsibility.

## Entity model

`PersonaEntity` is the source of truth. It emits seven event types and accumulates a full change log. `PersonaView` projects every event into a row for the UI. `PersonaChangeConsumer` subscribes to entity events to validate changes and manage the monitoring window. `PersonaWatchWorkflow` both reads (`getChange`) and writes (`markActive`, `recordRevalidation`, `openMonitoringWindow`) on the entity. The relationship between `PersonaAgent` and `AgentResponse` is "returns" — the agent's task result is the response record, whether answering a probe or a live user query.

## Defence-in-depth governance flow

For any persona that becomes active, the change passed through:

1. **Configuration gate** — `PersonaValidator` blocked adversarial instructions, missing fields, and non-allowlisted models before any activation event was written.
2. **PersonaAgent** — one model call, definition rebuilt from the validated snapshot.
3. **Behavioral revalidation** — `BehavioralRevalidator` scored probe responses against expected behavioral signatures; a FAIL badge surfaces in the UI.
4. **Human-on-the-loop monitoring** — the first 5 minutes of live conversations are forwarded to the operator channel so a human can observe real behavior.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
