# Architecture — fun-facts

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `FactEndpoint` accepts a topic submission, writes a `FactRequested` event onto `FactRequestEntity`, and immediately starts `FactRequestWorkflow`. The workflow's `generateStep` calls `FactGeneratorAgent` — the single AutonomousAgent — with the topic string as `TaskDef.instructions(...)`. The agent returns a `FactCollection`, which the workflow writes back to the entity via `recordCollection`. `FactRequestView` projects every entity event into a read-model row; `FactEndpoint` serves the read model to the UI over REST and SSE.

There is no sanitizer, no Consumer subscription step, and no guardrail in this baseline. The graph is intentionally minimal — the fewest components needed to demonstrate the single-agent pattern end-to-end.

## Interaction sequence

The sequence traces the happy path (J1). The flow has two moments of notable latency:

1. The `generateStep` call to the LLM — bounded by the 60 s step timeout. In practice a short topic string returns in 3–10 s.
2. The SSE event from `FactRequestEntity` to the browser — sub-second once the entity event is written.

There is no polling loop in this baseline (unlike blueprints that use a Consumer for pre-processing). The workflow advances directly from entity creation to agent call.

## State machine

Four states. The paths:

- The happy path is `PENDING → GENERATING → GENERATED`.
- A failure during the agent call lands in `FAILED`. A `FAILED` request's prior data (topic, requestedBy, requestedAt) is preserved on the entity.
- `PENDING → FAILED` can also occur if a validation error is caught before the workflow starts (e.g., a blank topic string caught at the endpoint layer — though the entity itself never transitions to FAILED in that scenario since no entity is created).
- There is no retry or revision state. If the agent fails all iterations, the error step fires immediately.

## Entity model

`FactRequestEntity` is the source of truth. It emits four event types. `FactRequestView` projects every event into a row used by the UI. `FactRequestWorkflow` both reads and writes on the entity — `markGenerating` before the agent call and `recordCollection` or `fail` after. The relationship between `FactGeneratorAgent` and `FactCollection` is "returns" — the agent's task result is the collection record.

## Minimal-skeleton properties

This blueprint deliberately omits the governance components present in fuller examples:

- No `Consumer` — the workflow starts from the endpoint directly, which is appropriate when there is no pre-processing step that needs to run asynchronously before the agent is invoked.
- No guardrail — the output schema is enforced by the Akka task's `resultConformsTo(FactCollection.class)` declaration, which handles parse failures at the framework level.
- No on-decision eval — the use case is informational content where automated evidence scoring would not add value for the end user.

Deployers who need any of these components can add them without restructuring the existing entity, workflow, or agent.
