# Architecture — sales-roleplay-coach

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`CoachEndpoint` is the primary entry point. It writes a `ScenarioSubmitted` event to `ScenarioQueue` (event-sourced for audit). A `ScenarioRequestConsumer` subscribes to that queue and starts a `SessionWorkflow` per submission. The workflow orchestrates two agents — `BuyerSimulatorAgent` opens the session and responds to each rep pitch, `CoachAgent` scores each pitch — with a deterministic guardrail check between the rep's submission and the buyer's response. Each step emits an event on `SessionEntity`. `SessionsView` projects those events into the read model the UI streams via SSE.

Two TimedActions operate in the background: `ScenarioSimulator` drips a canned practice scenario every 90 seconds so the UI has content when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any coached turn that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the rep's first turn is sent back for revision and the second turn earns an `ACCEPT`. The submission → guardrail → buyer response → coach scoring chain is explicit. Each turn's events are written to `SessionEntity` before the workflow pauses to wait for the next rep input, so the UI's per-turn timeline reconstructs the conversation exactly.

## State machine

The session moves between two transient states (`PITCHING`, `EVALUATING`) and two terminal states (`PASSED`, `EXHAUSTED`). The self-loop on `PITCHING` represents the guardrail-blocked path: a prohibited-content turn sends the workflow back to wait for a revised rep submission. The `EVALUATING → PITCHING` transition is the `REVISE` path — taken when the coach returns `REVISE` and the turn count is still below the ceiling. `EXHAUSTED` is reached only when the ceiling is hit; both terminal states preserve every turn and every coaching verdict on the entity.

## Entity model

`SessionEntity` is the system's source of truth; every transition writes one of eight event types. `ScenarioQueue` is the audit log of practice requests. `SessionsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 90 s for `openBuyerStep`, `buyerResponseStep`, and `coachStep`; the guardrail step is in-process and effectively instant. The `repTurnStep` wait state uses the same 90 s timeout — if the rep submits no turn, the workflow fails over to `exhaustStep`.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxTurns` (default 5). A wall-clock guard can be added by lifting `maxTurns` into a duration; outside the scope of the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(exhaustStep)` — any unrecoverable agent failure ends in `EXHAUSTED`, not in a hung workflow.
- Idempotency: `CoachEndpoint.createSession` deduplicates on `(buyerPersona, product, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(sessionId, turnNumber)` so a tick that fires twice for the same turn is a no-op.
- Per-turn accounting: every turn is appended to `Session.turns` with its own rep text, guardrail verdict, buyer response (once produced), and coaching verdict (once produced). The `turnNumber` is monotonic across the whole session, including guardrail-blocked turns that never reached the buyer simulator.
