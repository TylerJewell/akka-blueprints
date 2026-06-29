# Architecture — bug-assistant

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `BugEndpoint` accepts a submission and writes a `BugSubmitted` event onto `BugEntity`. The `TicketSyncConsumer` Consumer subscribes, builds simulated ticket metadata (minting a deterministic ticket key, attaching priority as a label), and writes the enriched metadata back via `attachEnriched`. The same Consumer then starts a `BugWorkflow` instance. The workflow's `investigateStep` calls `BugResolutionAgent` — the single AutonomousAgent — with the bug details as `TaskDef.instructions(...)`. During its task, the agent issues `search_web` and `read_ticket` tool calls (backed by in-process seed data), then issues a `write_ticket` tool call. Before that call executes, the agent's `before-tool-call` guardrail (`TicketWriteGuardrail`) validates the arguments. Once the call passes, the workflow receives the `Resolution`, writes `ResolutionRecorded`, and calls `closeStep` to finalize the entity. `BugView` projects every entity event into a read-model row; `BugEndpoint` serves the read model to the UI over REST and SSE.

The graph deliberately has no second agent. The `TicketWriteGuardrail` is a pure validation class — not an LLM. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Note the two distinct pauses:

1. The `TicketSyncConsumer` subscription lag between `BugSubmitted` and `TicketEnriched` — sub-second in normal operation.
2. The `awaitEnrichedStep` polling loop inside the workflow — polls `BugEntity` every 1 s up to its 15 s timeout, advancing as soon as `bug.ticket().isPresent()` returns true.

The agent call is bounded by `investigateStep`'s 90 s timeout to accommodate multi-turn tool-call sequences. The `closeStep` is synchronous and finishes immediately — no external service, no LLM call.

## State machine

Five states. The interesting paths:

- The happy path is `SUBMITTED → ENRICHED → INVESTIGATING → RESOLVED`.
- Two failure transitions land in `FAILED`: an enrichment error during `SUBMITTED`, and an agent error (or guardrail-budget exhaustion) during `INVESTIGATING`. A `FAILED` bug's prior data is preserved on the entity — the UI shows the partial state.
- There is no `VERIFIED` or `DEPLOYED` state. The resolution is a proposal; what happens next is outside the system's scope.

## Entity model

`BugEntity` is the source of truth. It emits five event types. `BugView` projects every event into a row used by the UI. `TicketSyncConsumer` subscribes to entity events to build the enriched ticket metadata. `BugWorkflow` both reads (`getBug`) and writes (`markInvestigating`, `recordResolution`, `close`, `fail`) on the entity. The relationship between `BugResolutionAgent` and `Resolution` is "returns" — the agent's task result is the resolution record.

## Governance flow

For any resolution that lands in the entity log, the bug passed through:

1. **TicketSyncConsumer enrichment** — the ticket is fetched before the agent ever runs, so the agent has real metadata to reason about.
2. **BugResolutionAgent** — one model call, one structured output, tool calls for research.
3. **before-tool-call guardrail** — empty resolution bodies, unknown status values, and mismatched ticket ids are caught before the write tool executes.

Each step is independent. Removing the guardrail opens an explicit gap — the agent could write to the wrong ticket or submit an incomplete resolution body — that the other steps do not cover.
