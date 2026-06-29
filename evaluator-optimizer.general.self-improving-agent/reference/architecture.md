# Architecture — self-improving-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`AgentEndpoint` is the entry point. It writes a `TaskSubmitted` event to `TaskQueueEntity` (event-sourced for audit). A `Consumer` subscribes to that queue and starts an `ImprovementWorkflow` once a batch is ready. The workflow alternates two agents — `ExecutorAgent` processes tasks under the current config, `OptimizerAgent` analyzes the resulting performance records and proposes a revision — with a regression attestation step between proposal and application. Each step emits an event on `AgentConfigEntity`. `ConfigView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `TaskSimulator` drips a sample task every 60 seconds so the UI is not empty when first loaded; `AttestationSampler` ticks every 30 seconds and records an `EvalRecorded` event for any revision cycle that has been attested but not yet sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first revision fails attestation and the second revision passes. The Executor → Optimizer → attestation alternation is explicit. Each cycle's events are written to `AgentConfigEntity` before the next step begins so the UI's per-cycle timeline reconstructs the improvement loop exactly.

## State machine

The agent configuration moves between two transient states (`ACTIVE`, `IMPROVING`) and two terminal states (`APPLIED`, `REVISION_REJECTED`). The self-loop on `IMPROVING` represents the failed-attestation path: a revision that does not beat the baseline sends the workflow back to `proposeStep` with the attestation failure attached. `REVISION_REJECTED` is reached when the ceiling is hit; both terminal states preserve every revision cycle and every attestation verdict on the entity.

## Entity model

`AgentConfigEntity` is the system's source of truth; every transition writes one of six event types. `TaskQueueEntity` is the audit log of submitted tasks. `ConfigView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 90 s for `executeStep`, `proposeStep`, and `attestStep`; the evaluate step is in-process and effectively instant.
- Workflow-wide bound: the workflow bounds itself with `maxRevisions` (default 3). Each failed attestation consumes one revision slot; the ceiling is checked before entering `proposeStep` for the next cycle.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REVISION_REJECTED`, not in a hung workflow.
- Idempotency: `AgentEndpoint.submitTask` deduplicates on `(instruction, submittedBy)` over a 10 s window. `AttestationSampler` deduplicates on `(configId, cycleNumber)` so a tick that fires twice for the same cycle is a no-op.
- Attestation determinism: the reference task set used in `attestStep` is the first three tasks from `sample-events/tasks.jsonl`, fixed at workflow start. The baseline score comes from the initial `executeStep` run so the delta is computed against actual observed performance, not a stored historical figure.
- Monitor stream: `/api/configs/monitor/sse` filters the `AgentConfigEntity` event stream to terminal transitions only. Connecting clients receive `RevisionApplied` and `RevisionRejected` events as they occur, carrying the full `AgentConfig` state.
