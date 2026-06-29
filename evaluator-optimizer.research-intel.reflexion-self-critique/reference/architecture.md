# Architecture — reflexion-self-critique

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Below is the narrative the tab also shows beside each diagram.

## Component graph

`ResearchEndpoint` is the entry point. It writes a `QuestionSubmitted` event to `QueryQueue` (event-sourced for audit). A `Consumer` subscribes to that queue and starts a `ReflexionWorkflow` per submission. The workflow alternates two agents — `ActorAgent` retrieves documents and produces a cited answer, `ReflexionAgent` scores it and produces a verbal reinforcement note — with a deterministic citation-floor guardrail between them. Each step emits an event on `QueryEntity`. `QueriesView` projects those events into the read model the UI streams via SSE.

Two TimedActions run alongside: `QuerySimulator` drips a sample research question every 60 seconds so the UI is never empty on first load; `ReflexionSampler` ticks every 30 seconds and records a `ReflexionRecorded` event for any reflected attempt that has not yet been sampled.

## Interaction sequence

The sequence diagram traces a J1-style convergence where the first answer passes the citation guardrail but receives a `RETRY` verdict from the reflexion agent, and the second answer — now conditioned on the verbal reinforcement note — is accepted with `PASS`. The actor's document retrieval tool call is shown explicitly. Each attempt's events are written to `QueryEntity` before the next step begins, so the UI's per-attempt timeline reconstructs the full loop.

## State machine

The query moves between two transient states (`RESEARCHING`, `REFLECTING`) and two terminal states (`RESOLVED`, `EXHAUSTED`). The self-loop on `RESEARCHING` represents the citation-guardrail-blocked path: an under-cited answer sends the workflow back to `answerStep` with structured feedback. The `REFLECTING → RESEARCHING` transition is the `RETRY` path — taken when the reflexion agent returns `RETRY` and the attempt count is still below the ceiling. `EXHAUSTED` is reached only when the ceiling is hit with no `PASS`; both terminal states preserve every attempt and every reflexion note on the entity.

## Entity model

`QueryEntity` is the system's source of truth; every transition writes one of seven event types. `QueryQueue` is the audit log of submissions. `QueriesView` is the only read-side projection — the UI never queries the entity directly.

## Concurrency & timeouts

- Per-step timeout: 60 s for `answerStep` and `reflectStep`; the citation guardrail step is in-process and effectively instant.
- Workflow-wide deadline: the workflow has no wall-clock deadline; the loop bounds itself with `maxAttempts` (default 4). A wall-clock guard could be added by lifting `maxAttempts` into a duration.
- Default step recovery: `maxRetries(2).failoverTo(exhaustStep)` — any unrecoverable agent failure ends in `EXHAUSTED`, not in a hung workflow.
- Idempotency: `ResearchEndpoint.submit` deduplicates on `(questionText, submittedBy)` over a 10 s window. `ReflexionSampler` deduplicates on `(queryId, attemptNumber)` so a tick that fires twice for the same attempt is a no-op.
- Per-attempt accounting: every attempt is appended to `Query.attempts` with its own answer, citation guardrail verdict, and (once produced) reflexion. The `attemptNumber` is monotonic across the whole loop, including citation-blocked attempts that never reached the reflexion agent.
- Verbal memory propagation: on `RETRY_ANSWER`, the full `ReflexionNote` is passed as a structured input to `ActorAgent`. The agent serializes this into its prompt context as prior reasoning, conditioning subsequent retrieval queries and synthesis.
