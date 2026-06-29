# Architecture — pr-reviewer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ReviewEndpoint` is the entry point. It writes a `PrSubmitted` event to `PrQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ReviewWorkflow` per submission. The workflow alternates two agents — `ReviewerAgent` produces a feedback draft, `AlignmentAgent` checks it — with a deterministic personal-critique guardrail scan between them. Each step emits an event on `ReviewEntity`. `ReviewsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `PrSimulator` drips a sample PR diff every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any alignment-checked attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first feedback draft is sent back for revision and the second draft is approved. The Reviewer → guardrail → Alignment alternation is explicit. Each attempt's events are written to `ReviewEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The review moves between two transient states (`REVIEWING`, `CHECKING_ALIGNMENT`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `REVIEWING` represents the guardrail-blocked path: a draft with personal-critique language sends the workflow back to `reviewStep` with structured feedback. The `CHECKING_ALIGNMENT → REVIEWING` transition is the `REVISE` path — taken when the alignment agent returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every feedback draft and every alignment check on the entity.

## Entity model

`ReviewEntity` is the system's source of truth; every transition writes one of seven event types. `PrQueue` is the audit log of PR submissions. `ReviewsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `reviewStep` and `alignmentStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `ReviewEndpoint.submit` deduplicates on `(diffText hash, submittedBy)` over a 10 s window. `EvalSampler` deduplicates on `(reviewId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Review.attempts` with its own feedback draft, guardrail verdict, and (once produced) alignment check. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the alignment agent.
