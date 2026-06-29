# Architecture — Dynamic Route Agent

One generic `Agent` is configured per request. The same code path serves a SUMMARIZE route and a TRANSLATE route; the route picks the instruction text and capability built into the per-request `AgentSetup`. Two guardrails bracket the agent call. The four diagrams below are the mermaid source the generated system renders on the Architecture tab; they must inherit the Lesson 24 state-label CSS overrides and theme variables.

## Component graph

The flow is synchronous and single-agent. `RouteEndpoint` receives a submit, runs the before-agent-invocation guardrail (G1), records `RequestSubmitted` on `RequestEntity`, invokes `DynamicAgent`, runs the before-agent-response guardrail (G2), then records `RequestCompleted` or `RequestBlocked`. `RequestEntity` events project into `RequestsView`, which the browser streams over SSE. `RequestSimulator` drips canned requests through the same submit path so the UI shows activity without user input. See the `flowchart TB` in `PLAN.md`.

## Interaction sequence

The primary journey is submit -> validate config -> persist submitted -> run agent -> validate output -> persist completed, with SSE updates pushed to the browser at each state change. The `sequenceDiagram` in `PLAN.md` carries `Note over` blocks marking where each guardrail fires.

## State machine

`RequestStatus` has three states: `SUBMITTED`, `COMPLETED`, `BLOCKED`. A request enters `SUBMITTED` on submit. It moves to `COMPLETED` when the agent output passes G2, or to `BLOCKED` when either G1 (before the agent runs) or G2 (after it runs) rejects. `COMPLETED` and `BLOCKED` are terminal. The `stateDiagram-v2` in `PLAN.md` is the source.

## Entity model

`RequestEntity` is the single event-sourced entity; it emits `RequestSubmitted`, `RequestCompleted`, and `RequestBlocked`. `RequestsView` projects the entity into a `RequestRecord` row the UI queries and streams. The `erDiagram` in `PLAN.md` is the source.
