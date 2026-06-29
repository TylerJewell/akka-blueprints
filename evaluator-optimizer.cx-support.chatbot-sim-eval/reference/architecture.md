# Architecture — chatbot-sim-eval

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SimulationEndpoint` is the entry point. It writes a `ScenarioSubmitted` event to `ScenarioQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `SimulationWorkflow` per submission. The workflow alternates three agents — `SimulatedUserAgent` opens and replies as the customer, `ChatbotAgent` answers each customer turn, and after the dialogue concludes `EvaluatorAgent` scores the full transcript — with a deterministic guardrail check after each chatbot reply. Each step emits an event on `SimulationEntity`. `SimulationsView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `ScenarioSimulator` drips a sample scenario every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any completed simulation that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the simulated user signals resolution after the second chatbot reply. The SimulatedUserAgent → ChatbotAgent → guardrail alternation is explicit. Each turn's events are written to `SimulationEntity` before the next step begins so the UI's per-turn timeline reconstructs the dialogue exactly. After `ConversationConcluded` is emitted, `EvaluatorAgent` runs once over the full transcript and emits `EvalVerdictRecorded`.

## State machine

The simulation moves through two transient states (`RUNNING`, `EVALUATING`) and two terminal states (`PASSED_EVALUATION`, `FAILED_EVALUATION`). The self-loop on `RUNNING` represents each full turn cycle: user turn → chatbot reply → guardrail check → next user reply. The transition to `EVALUATING` fires when the simulated user signals resolution or the turn ceiling is hit. Both terminal states preserve the full transcript, every guardrail flag, and the evaluator's findings on the entity.

## Entity model

`SimulationEntity` is the system's source of truth; every transition writes one of six event types. `ScenarioQueue` is the audit log of submissions. `SimulationsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 90 s for `openDialogueStep`, `chatbotStep`, and `userReplyStep`; 120 s for `evaluateStep`. The guardrail step is in-process and effectively instant.
- Workflow-wide ceiling: the loop bounds itself with `maxTurns` (default 10). A wall-clock guard could be added by lifting `maxTurns` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(failStep)` — any unrecoverable agent failure ends in `FAILED_EVALUATION`, not in a hung workflow.
- Idempotency: `SimulationEndpoint.submit` deduplicates on `(personaKey, issueDescription)` over a 10 s window. `EvalSampler` deduplicates on `simulationId` so a tick that fires twice for the same completed simulation is a no-op.
- Per-turn accounting: every dialogue turn is appended to `Simulation.turns` with its `UserTurn`, its `AssistantTurn` (including the guardrail flag), and the turn number. The evaluator receives the full list at evaluation time.
- CI gate: `GET /api/simulations/ci-gate` is a synchronous read from `SimulationsView` with no state mutation. It returns immediately if all requested simulations are in terminal states; the caller is responsible for polling if any are still `RUNNING` or `EVALUATING`.
