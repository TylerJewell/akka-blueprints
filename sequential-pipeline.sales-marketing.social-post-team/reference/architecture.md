# Architecture

The system is a sequential pipeline: a workflow drives a fixed order of agent steps, each consuming the prior step's typed output, with two guardrails and a human gate wired into the path. Everything runs in one process; the web-search and browse tools are in-process endpoints, so the sample runs out of the box. The four diagrams below are the source the Architecture tab renders.

## Component graph

Every component, colored by Akka primitive. Solid arrows are synchronous commands, dashed arrows are event subscriptions, dotted arrows are scheduled ticks. See the `flowchart TB` block in `PLAN.md`.

`ConceptSimulator` drips canned concepts into `ConceptQueue`; `PostEndpoint` also enqueues concepts from real submissions. `ConceptConsumer` reacts to each `ConceptQueued` event and starts one `PostWorkflow`. The workflow calls `ResearchAgent` (which reaches the `WebTools` endpoint), `CopyAgent`, and `VisualDirectorAgent` in order, writing results into `PostEntity`. `PostsView` projects `PostEntity` events into the read model the UI streams. `StalePostMonitor` ticks over the view and escalates stuck posts.

## Interaction sequence

The primary journey: submit a concept, the pipeline researches and composes, the post waits in `AWAITING_APPROVAL`, a marketer approves, the post publishes. The `Note over` blocks mark the paused approval state and the terminal published state. See the `sequenceDiagram` block in `PLAN.md`.

## State machine

`PostEntity` lifecycle: `RESEARCHING` → `COMPOSING` → `AWAITING_APPROVAL` → `APPROVED` → `PUBLISHED`, with `REJECTED` (from a failed brand check or a human reject) and `ESCALATED` (from the stale-post monitor) as terminal branches. See the `stateDiagram-v2` block in `PLAN.md`. The Architecture tab must apply the Lesson 24 CSS overrides so state names and transition labels render legibly.

## Entity model

`ConceptQueue` records inbound concepts and starts `PostEntity` instances; `PostEntity` projects into `PostsView`. See the `erDiagram` block in `PLAN.md`.

## Concurrency

Agent-calling steps carry explicit 60-second timeouts; the await-approval step self-schedules a 5-second resume timer and re-polls the entity. The workflow id is the post id, so re-delivery of a queued concept reuses the same workflow instance. A failed brand check transitions the post to `REJECTED` with no publish side effect. Full notes in `PLAN.md`.
