# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab.

## Component graph

A request enters either from `ResearchEndpoint` (a client POST) or from `RequestSimulator` (a TimedAction that drips canned requests every 30s). Both write to `InboundRequestQueue`, an event-sourced entity that buffers requests as `InboundRequestQueued` events. `RequestConsumer` subscribes to those events and starts one `ResearchWorkflow` per request with a fresh UUID. The workflow drives the two agents and records every transition on `BriefEntity`. `BriefsView` projects the entity's events into a read model that `ResearchEndpoint` queries and streams over SSE. `AppEndpoint` serves the single-file UI.

## Interaction sequence

The primary journey is plan → analyze → synthesize → sanitize. The supervisor plans the sector list; the worker is invoked once per sector; the supervisor synthesizes a brief from the collected findings. The grounding guardrail runs on the synthesis before the result is accepted — a `groundingScore` below the bar diverts the brief to a terminal `BLOCKED` state instead of releasing it. A brief that clears the bar is sanitized (disclaimer appended) and marked `COMPLETED`.

## State machine

`BriefEntity` moves PLANNED → ANALYZING → SYNTHESIZED → SANITIZED → COMPLETED on the success path, with a single branch to BLOCKED when grounding fails. BLOCKED and COMPLETED are terminal. The status enum is closed — no other states exist.

## Entity model

`BriefEntity` is the only lifecycle entity; `InboundRequestQueue` is a thin buffer. The brief aggregates a list of `SectorFinding` (one per analyzed sector) and emits the six `Brief*` events. `BriefsView` holds one row per brief, keyed by id, and is the only read surface the API and UI touch.

## Mermaid theming note

The Architecture tab's `stateDiagram-v2` needs the CSS overrides from Lesson 24: state-box labels must be forced white (`color`/`fill` on the `g.statediagram-state .label` DOM paths) and edge-label `foreignObject` elements must be `overflow: visible`, or the state names render black-on-black and transition labels clip. The generated `index.html` carries both the CSS block and the matching `mermaid.initialize` theme variables (`nodeTextColor`, `stateLabelColor`, `transitionLabelColor: #cccccc`).
