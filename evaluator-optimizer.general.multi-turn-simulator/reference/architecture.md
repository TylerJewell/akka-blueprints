# Architecture — multi-turn-simulator

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SimulationEndpoint` is the entry point. It writes a `ScenarioSubmitted` event to `ScenarioQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SimulationWorkflow` per submission. The workflow alternates two agents — `ActorAgent` generates the next user utterance, `TargetAgentProxy` forwards it to the configured target endpoint, and `EvaluatorAgent` scores the response — with a deterministic guardrail check between the proxy and the evaluator. Each step emits events on `SessionEntity`. `SessionsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `ScenarioSimulator` drips a canned scenario every 90 seconds so the UI is not empty when first loaded; `DriftSampler` ticks every 45 seconds and records a `DriftEvalRecorded` event for any scored turn that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style clean session where two turns complete before the ActorAgent signals goal completion. The Actor → proxy → guardrail → Evaluator alternation is explicit. Each turn's events are written to `SessionEntity` before the next step begins so the UI's per-turn timeline reconstructs the dialog exactly.

## State machine

The session moves through one transient state (`RUNNING`) and three terminal states (`COMPLETED`, `FLAGGED`, `HALTED`). The self-loop on `RUNNING` represents both normal progression (each scored turn increments the counter) and the guardrail-blocked path (a policy-violating response synthesizes a POLICY_VIOLATION verdict and the loop continues). `FLAGGED` is reached when the session closes with any drift or policy-violation flag set. `HALTED` is reached when the turn ceiling is hit or a step fails unrecoverably. All terminal states preserve every turn record and every verdict on the entity.

## Entity model

`SessionEntity` is the system's source of truth; every turn transition writes one of seven event types. `ScenarioQueue` is the audit log of scenario submissions. `SessionsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `actorStep`, `targetStep`, and `scoreStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxTurns` (default 10). A wall-clock guard could be added by converting `maxTurns` to a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(haltStep)` — any unrecoverable agent or network failure ends in `HALTED`, not in a hung workflow.
- Idempotency: `SimulationEndpoint.submit` deduplicates on `(persona, goal, submittedBy)` over a 10 s window. `DriftSampler` deduplicates on `(sessionId, turnNumber)` so a tick that fires twice for the same turn is a no-op.
- Per-turn accounting: every turn is appended to `Session.turns` with its own utterance, response, guardrail verdict, and (once scored) turn verdict. The `turnNumber` is monotonic across the whole session, including guardrail-blocked turns that synthesized a POLICY_VIOLATION verdict rather than calling the EvaluatorAgent.
- TargetAgentProxy: when `multi-turn-simulator.simulation.target-url = "stub"`, the proxy returns deterministic canned responses from `src/main/resources/stub-responses/target.json` keyed on `(persona, turnNumber)`. No external service is needed to run the blueprint.
