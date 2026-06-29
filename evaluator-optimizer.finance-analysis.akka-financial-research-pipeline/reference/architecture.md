# Architecture — financial-research-pipeline

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ResearchEndpoint` is the entry point. It writes a `QuerySubmitted` event to `QueryQueue` (event-sourced for audit and replay). A `Consumer` subscribes to that queue and starts a `ResearchWorkflow` per submission. The workflow drives five agents in sequence: `PlannerAgent` decomposes the query; `SearchAgent` retrieves source material for each sub-task; `AnalystAgent` synthesises the bundles into analysis sections; a pure-function sanitiser step scrubs disallowed financial identifiers before the writer sees them; `WriterAgent` produces the prose draft; a pure-function guardrail step checks the word count; `VerifierAgent` quality-checks the draft and either approves or revises. Each step emits events on `ResearchEntity`. `ReportsView` projects those events into the read model the UI streams via SSE.

Two TimedActions sit alongside: `QuerySimulator` drips a sample query every 90 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 45 seconds and records an `EvalRecorded` event for any verified draft that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first draft exceeds the word ceiling (guardrail blocks it), the revised draft is sent to the verifier and receives a `REVISE` verdict, and the third draft is approved. The PlannerAgent → SearchAgent (×3) → AnalystAgent → sanitiser → WriterAgent → guardrail → VerifierAgent ordering is explicit. The compliance parking step is shown after the final `APPROVE` — the workflow parks and resumes on the reviewer's HTTP call.

## State machine

The report moves through three planning states (`PLANNING`, `RESEARCHING`, `ANALYSING`) and two draft-iteration states (`WRITING`, `VERIFYING`) before reaching either a parked state (`AWAITING_COMPLIANCE`) or a rejection terminal (`REJECTED_FINAL`). The self-loop on `WRITING` represents the guardrail-blocked path. The `VERIFYING → WRITING` transition is the `REVISE` path, taken when the verifier returns `REVISE` and the draft count is still below the ceiling. `APPROVED` is reached only after a compliance reviewer explicitly calls the approval endpoint.

## Entity model

`ResearchEntity` is the source of truth; every pipeline transition writes one of twelve event types. `QueryQueue` is the audit log of submitted queries. `ReportsView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 90 s for all agent-calling steps; 5 s for the sanitiser and guardrail steps, which are pure-function and in-process.
- Compliance parking: the workflow parks after a verifier `APPROVE`. While parked it holds no thread; it resumes on the reviewer's HTTP call. There is no wall-clock deadline on the parked step in the default blueprint; a deployer can add a timeout that auto-rejects after a configurable interval.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`.
- Search fan-out: sub-task searches run sequentially. The `searchStep` loops over the plan's sub-tasks; each iteration waits for the previous `SourceBundle` before starting the next.
- Idempotency: `ResearchEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 10 s window. `EvalSampler` deduplicates on `(reportId, draftNumber)`.
- Sanitiser step: pure-function; applies the regex list from `financial-research-pipeline.sanitiser.disallowed-patterns`. The default list is empty (no-op), and the `SanitizerLogRecorded` event is still emitted with `changed = false`.
