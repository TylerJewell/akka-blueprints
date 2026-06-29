# Architecture — self-improving-deep-researcher

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ResearchEndpoint` is the entry point. It writes a `QuerySubmitted` event to `QueryQueue` (event-sourced for replay and audit). A `Consumer` subscribes to that queue and starts a `ResearchSessionWorkflow` per submission. The workflow orchestrates three agents in sequence: `ResearchAgent` synthesizes a report from the query and current memory blocks, `EvaluatorAgent` scores the report against a quality rubric, and — after the research loop ends — `PromptRewriterAgent` updates the `PromptMemory` blocks to improve future sessions. Each step emits an event on `SessionEntity`. `SessionsView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `QuerySimulator` drips a sample research query every 90 seconds so the UI is not empty when first loaded; `DriftSampler` ticks every 60 seconds, hashes the live `PromptMemory`, and emits a `PromptDriftRecorded` event whenever the fingerprint changes.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first research attempt fails the quality rubric and the second is accepted. The ResearchAgent → EvaluatorAgent alternation is explicit, with QualityNotes flowing from the evaluator back into the refine call. After the `SessionAccepted` event, `PromptRewriterAgent` runs unconditionally and its `MemoryDiff` is stored as a `MemoryUpdated` event before the SSE update reaches the UI.

## State machine

The session moves through `RESEARCHING` and `EVALUATING` as transient states. The `EVALUATING → RESEARCHING` arc represents the `REFINE` path — taken when the evaluator returns `REFINE` and the attempt count is still below the ceiling. Both terminal states (`ACCEPTED`, `MAX_ATTEMPTS_REACHED`) transition to a `MEMORY_REWRITING` state that ends when the `MemoryUpdated` event is emitted. This ensures the prompt-rewriter step is always observable in the session lifecycle, not a silent side-effect.

## Entity model

`SessionEntity` is the source of truth; every transition writes one of eight event types. `QueryQueue` is the submission audit log. `SessionsView` is the only read-side projection — the UI never queries the entity directly. The `MemoryUpdated` event carries the full `MemoryDiff`, making the entity the durable record of every prompt change and which session triggered it.

## Concurrency and timeouts

- Per-step timeout: 90 s for `researchStep`, `refineStep`, `evaluateStep`, and `rewriteMemoryStep`. Research synthesis and memory rewriting involve multi-step LLM reasoning; the extended timeout is required.
- Default step recovery: `maxRetries(2).failoverTo(maxAttemptsStep)` — any unrecoverable agent failure ends in `MAX_ATTEMPTS_REACHED`. The memory rewrite step is outside the recovery failover scope; its own `maxRetries(2)` guard ensures transient failures do not skip the rewrite.
- Idempotency: `ResearchEndpoint.submit` deduplicates on `(topic, requestedBy)` over a 15 s window. `DriftSampler` deduplicates on `(oldFingerprint, newFingerprint)` so a duplicate tick does not inflate the drift event count.
- Memory store concurrency: the `rewriteMemoryStep` writes to `governance/prompt-memory.json` as a single atomic file replacement. If two sessions finish close together, the second write wins; partial diffs do not accumulate. In a high-concurrency deployment this file should be replaced with a transactional KV store.
- Attempt accounting: every attempt is appended to `Session.attempts` with its own report and, once produced, its evaluation. The `attemptNumber` is monotonic. The evaluator always sees the attempt-number in context so it can note regressions across refinements.
