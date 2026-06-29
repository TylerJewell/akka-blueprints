# Architecture — memory-eval-loop

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`SessionEndpoint` is the entry point. It writes a `SessionCreated` event to `SessionEntity` (event-sourced for full audit trail). A `Consumer` subscribes to that entity and starts an `AnswerWorkflow` per session turn. The workflow first queries `MemoryView` to retrieve relevant context for the user, then alternates two agents — `AnswerAgent` produces a cited answer, `ScorerAgent` grades it — until convergence or the retry ceiling. After the session reaches a terminal state, the workflow applies the PII sanitizer and writes new fact fragments back to `MemoryStore`. `SessionView` projects `SessionEntity` events into the read model the UI streams via SSE.

Three TimedActions sit alongside: `QuestionSimulator` drips a sample question every 60 seconds so the UI is not empty when first loaded; `EvalSampler` ticks every 30 seconds and records an `EvalRecorded` event for any scored attempt not yet sampled; `DriftWatcher` ticks every 10 minutes and computes a rolling pass-rate and average-score across recent turns to detect quality drift as memory accumulates.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first answer attempt scores `IMPROVE` and the second is accepted. The retrieve → answer → score alternation is explicit. Each attempt's events are written to `SessionEntity` before the next step begins, so the UI's per-attempt timeline reconstructs the loop exactly. The memory write happens after `SessionAccepted`, never before.

## State machine

The session turn moves between two transient states (`ANSWERING`, `SCORING`) and two terminal states (`ACCEPTED`, `REJECTED_FINAL`). There is no guardrail self-loop in this blueprint — the memory retrieval step is non-blocking and always produces a result (even an empty list). The `SCORING → ANSWERING` transition is the `IMPROVE` path, taken when the scorer returns `IMPROVE` and the attempt count is still below the ceiling. `REJECTED_FINAL` is reached when the ceiling is hit; both terminal states preserve every attempt and every score on the entity.

## Entity model

`SessionEntity` is the system's source of truth for session turns; every transition writes one of seven event types. `MemoryStore` is the system's durable memory corpus; it emits two event types and is never read directly by the UI — `MemoryView` is the only read path. `SessionView` and `MemoryView` are the only read-side projections; the UI never queries entities directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `answerStep` and `scoreStep`; the retrieve step is a view query and uses the view client's default timeout.
- Workflow-wide deadline: the loop bounds itself with `maxAttempts` (default 3). A wall-clock guard could be added by mapping `maxAttempts` to a duration; out of scope for the default blueprint.
- Default step recovery: `maxRetries(2).failoverTo(rejectStep)` — any unrecoverable agent failure ends in `REJECTED_FINAL`, not in a hung workflow.
- Memory write ordering: `memoryWriteStep` executes only after the session entity is in a terminal state, ensuring the accepted answer text is available and the PII sanitizer runs on final content.
- EvalSampler idempotency: keyed on `(sessionId, attemptNumber)` — a tick that fires twice for the same attempt is a no-op on the entity side.
- DriftWatcher sentinel: writes drift check events to the fixed entity id `"drift-watch-singleton"` to keep drift history centralized without creating one entity per check.
- Memory retrieval on revision: the retrieve step runs at the start of every answer attempt, including revisions. If new memory entries were written by a concurrent session, the revised answer can cite them.
