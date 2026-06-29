# Architecture — evaluator-optimizer-workflow

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`EditorialEndpoint` is the entry point. It writes an `ArticleSubmitted` event to `SubmissionQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `HeadlineWorkflow` per submission. The workflow alternates two agents — `WriterAgent` produces a headline draft, `ReviewerAgent` scores it — with a deterministic guardrail check between them. Each step emits an event on `HeadlineEntity`. `HeadlinesView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `BriefSimulator` drips a sample article brief every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any reviewed attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first draft is sent back for revision and the second draft is approved. The Writer → guardrail → Reviewer alternation is explicit. Each attempt's events are written to `HeadlineEntity` before the next step begins so the UI's per-attempt timeline reconstructs the loop exactly.

## State machine

The headline moves between two transient states (`DRAFTING`, `REVIEWING`) and two terminal states (`APPROVED`, `REJECTED_FINAL`). The self-loop on `DRAFTING` represents the guardrail-blocked path: an over-ceiling draft sends the workflow back to `draftStep` with structured feedback. The `REVIEWING → DRAFTING` transition is the `REVISE` path — taken when the reviewer returns `REVISE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached only when the ceiling is hit; both terminal states preserve every attempt and every review on the entity.

## Entity model

`HeadlineEntity` is the system's source of truth; every transition writes one of seven event types. `SubmissionQueue` is the audit log of submissions. `HeadlinesView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `draftStep` and `reviewStep`; the guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Idempotency: `EditorialEndpoint.submit` deduplicates on `(summary, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(headlineId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Headline.attempts` with its own draft, guardrail verdict, and (once produced) review. The `attemptNumber` is monotonic across the whole loop, including guardrail-blocked attempts that never reached the reviewer.
