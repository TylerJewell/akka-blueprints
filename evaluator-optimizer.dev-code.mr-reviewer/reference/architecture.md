# Architecture — mr-reviewer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab shows beside each diagram.

## Component graph

`ReviewEndpoint` is the entry point. It writes a `WebhookReceived` event to `MrQueue` (event-sourced for audit). A `WebhookConsumer` subscribes to that queue and starts a `ReviewWorkflow` per submission. The workflow runs two pure-function gatekeeping steps — the sanitizer and the output guardrail — around two agents: `ReviewerAgent` produces review commentary, `GateAgent` scores it. Each step emits an event on `MrEntity`. `MrView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `MrSimulator` drips a sample webhook every 90 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records a `ReviewEvalRecorded` event for any gated pass that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the gate returns `REFINE` on the first pass and `PASS` on the second. The sanitizer check, the reviewer, the output guardrail check, and the gate are all shown as separate participants to make the governance wiring explicit. Each pass's events are written to `MrEntity` before the next step begins, so the UI's per-pass timeline reconstructs the full loop trajectory.

## State machine

The MR moves through two transient states (`REVIEWING`, `GATE_CHECKING`) and three terminal states (`CI_PASS`, `CI_FAIL`, `SANITIZER_BLOCKED`). The self-loop on `REVIEWING` represents the output-guardrail-blocked path: commentary that reproduces a secret sends the workflow back to `reviewStep` with structured feedback. The `GATE_CHECKING → REVIEWING` transition is the `REFINE` path — taken when the gate returns `REFINE` and the pass count is still below the ceiling. `SANITIZER_BLOCKED` is a fast-exit path that bypasses all LLM calls. `CI_FAIL` is reached when the ceiling is hit or when `defaultStepRecovery` fires.

## Entity model

`MrEntity` is the system's source of truth; every transition writes one of eight event types. `MrQueue` is the audit log of inbound webhooks. `MrView` is the only read-side projection — the UI and the CI signal endpoint never query the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `reviewStep` and `gateStep`; the sanitizer and guardrail steps are in-process and effectively instant.
- Workflow-wide deadline: the workflow bounds itself with `maxPasses` (default 3). A wall-clock guard could be added; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(ciFailStep)` — any unrecoverable agent failure ends in `CI_FAIL`, not in a hung workflow.
- Idempotency: `ReviewEndpoint.submit` deduplicates on `(projectPath, mrIid)` over a 10 s window; a second webhook for the same MR returns the first `mrId` (200) rather than starting a new workflow.
- `EvalSampler` deduplicates on `(mrId, passNumber)` so a tick that fires twice for the same pass is a no-op on the entity side.
- **CI signal availability:** `MrEntity.ciSignal` is set atomically on the terminal event. `GET /api/mrs/{id}/ci-signal` returns `PENDING` until that event arrives; once set, the value is immutable.
