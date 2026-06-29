# Architecture — sdlc-technical-designer

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `DesignEndpoint` accepts a submission and writes a `DesignRequestSubmitted` event onto `DesignRequestEntity`. The `ContextLoader` Consumer subscribes, resolves the project context for the given `projectContextKey`, and writes the assembled context back via `attachContext`. The same Consumer then starts a `DesignWorkflow` instance. The workflow's `designStep` calls `DesignAgent` — the single AutonomousAgent — with the project context profile as `TaskDef.instructions(...)` and the feature description as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ProposalGuardrail`) validates each candidate response. Once a proposal passes, the workflow writes `ProposalRecorded` and runs `ProposalScorer` in `evalStep`. The score lands as `EvaluationScored`. `DesignView` projects every entity event into a read-model row; `DesignEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The on-decision evaluator (`ProposalScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note two distinct points where the system awaits something:

1. The `ContextLoader` subscription lag between `DesignRequestSubmitted` and `ContextLoaded` — sub-second in normal operation, since the context lookup is a local file read.
2. The `awaitContextStep` polling loop inside the workflow — polls `DesignRequestEntity` every 1 s up to its 15 s timeout, advancing as soon as `request.context().isPresent()` returns true.

The agent call itself is bounded by `designStep`'s 90 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Six states. The interesting paths:

- The happy path is `SUBMITTED → CONTEXT_LOADED → DESIGNING → PROPOSAL_RECORDED → EVALUATED`.
- Two failure transitions land in `FAILED`: a context-load error during `SUBMITTED`, and an agent error (or guardrail-exhaustion) during `DESIGNING`. A `FAILED` request's prior data is preserved on the entity — the UI shows the partial state.
- There is no `APPROVED` or `IMPLEMENTED` state. The proposal is advisory; the engineer reviews it and acts outside the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`DesignRequestEntity` is the source of truth. It emits six event types. `DesignView` projects every event into a row used by the UI. `ContextLoader` subscribes to entity events to assemble the context package. `DesignWorkflow` both reads (`getDesignRequest`) and writes (`markDesigning`, `recordProposal`, `recordEvaluation`, `fail`) on the entity. The relationship between `DesignAgent` and `DesignProposal` is "returns" — the agent's task result is the proposal record.

## Governance flow

For any proposal that lands in the entity log, the feature description passed through:

1. **ContextLoader** — the project context is resolved and attached before the agent ever sees the feature description, ensuring the design is grounded in the actual project environment.
2. **DesignAgent** — one model call, one structured output covering components, data model, API surface, and decision log.
3. **before-agent-response guardrail** — internally inconsistent proposals (broken componentId references, unknown primitive kinds, empty required sections) are caught before the response leaves the agent loop.
4. **On-decision evaluator** — every well-formed proposal still gets a 1–5 completeness score so the engineer knows which proposals need extra scrutiny before work begins.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
