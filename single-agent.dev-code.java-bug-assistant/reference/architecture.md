# Architecture — java-bug-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `BugEndpoint` accepts a submission and writes a `BugSubmitted` event onto `BugEntity`. The `BugNormalizer` Consumer subscribes, removes internal hostnames, stack-trace tokens, and credential patterns from the raw report, and writes the normalized form back via `attachNormalized`. `TicketSearcher` then picks up the `BugNormalized` event, queries the in-process `TicketStore` for related tickets, writes the results via `attachSearchResults`, and starts a `ResolutionWorkflow` instance. The workflow's `resolveStep` calls `BugResolutionAgent` — the single AutonomousAgent — with the search results as `TaskDef.instructions(...)` and the normalized bug report as a `TaskDef.attachment(...)`. The agent's `before-tool-call` guardrail (`TicketWriteGuardrail`) intercepts every ticket-write attempt and validates the candidate `ResolutionRecommendation` before allowing the call through. Once a recommendation passes, the workflow writes `RecommendationRecorded` and runs `RecommendationScorer` in `evalStep`. The score lands as `EvaluationScored`. `BugView` projects every entity event into a read-model row; `BugEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one component that talks to a model. The on-decision evaluator (`RecommendationScorer`) is a deterministic rule-based scorer — not an LLM. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct phases where the system waits on something:

1. The `BugNormalizer` subscription lag between `BugSubmitted` and `BugNormalized` — sub-second in normal operation.
2. The `TicketSearcher` subscription lag between `BugNormalized` and `SearchResultsAttached` — also sub-second since `TicketStore` is in-process.
3. The `awaitNormalizedStep` and `searchStep` polling loops inside the workflow — each polls `BugEntity` every 1 s, advancing once the relevant `Optional` field becomes present.

The agent call itself is bounded by `resolveStep`'s 60 s timeout. The `evalStep` is synchronous and finishes in milliseconds — no external service, no LLM call.

## State machine

Eight states, three terminal transitions. The interesting paths:

- The happy path is `SUBMITTED → NORMALIZED → SEARCHING → SEARCH_COMPLETE → RESOLVING → RECOMMENDATION_RECORDED → EVALUATED`.
- Three failure transitions land in `FAILED`: a normalizer error during `SUBMITTED`, a search error during `NORMALIZED`, and an agent error (or guardrail-exhaustion) during `RESOLVING`. A `FAILED` bug's prior data is preserved on the entity — the UI shows the partial state for the engineer.
- There is no `FIXED` or `CLOSED` state. The recommendation is advisory; the engineer reads it and takes action in their actual workflow outside the system. The blueprint stops at `EVALUATED`.

## Entity model

`BugEntity` is the source of truth. It emits eight event types. `BugView` projects every event into a row used by the UI. `BugNormalizer` subscribes to entity events to compute the normalized form. `TicketSearcher` subscribes to entity events to attach search results. `ResolutionWorkflow` both reads (`getBug`) and writes (`startResolving`, `recordRecommendation`, `recordEvaluation`, `fail`) on the entity. The relationship between `BugResolutionAgent` and `ResolutionRecommendation` is "returns" — the agent's task result is the recommendation record.

## Governance flow

For any recommendation that lands in the entity log, the bug report passed through:

1. **BugNormalizer** — internal hostnames, stack-trace tokens, and credential patterns are removed before the model sees anything.
2. **TicketSearcher** — relevant context is retrieved from the in-process ticket store and provided to the agent so it can cite sources rather than hallucinate.
3. **BugResolutionAgent** — one model call, one structured output.
4. **before-tool-call guardrail** — malformed writes (empty fix list, missing rationale, out-of-range confidence) are blocked before any bytes reach the ticket store.
5. **On-decision evaluator** — every well-formed recommendation still gets a 1–5 evidence score so the engineer knows which recommendations to trust at a glance.

Each step is independent. Removing one of them opens an explicit gap the others do not silently cover.
