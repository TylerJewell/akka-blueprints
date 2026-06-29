# Architecture — Meeting Prep Briefer

This system implements the delegation-supervisor-workers pattern. A supervisor agent decomposes a meeting request into research subtasks; worker agents carry them out against an in-process research source; a composer agent assembles the results. Everything runs in one Akka service — there are no external systems to stand up.

The four diagrams below are also rendered on the Architecture tab of the embedded UI. The mermaid source is authoritative in `PLAN.md`; this file narrates each one.

## Component graph

The flow starts at the `RequestQueue` entity, fed either by the `RequestSimulator` (a scheduled tick every 30s) or by a `POST /api/briefings`. The `BriefingRequestConsumer` reacts to each queued request and starts a `BriefingWorkflow`. The workflow drives the three agent phases, writes events to the `BriefingEntity`, and the `BriefingsView` projects those events into the read model the UI lists and streams. The `ResearchSource` endpoint is the in-process stand-in for an external research surface — the worker agents call it like a tool.

## Interaction sequence

A request enters the queue; the consumer starts the workflow. `planStep` asks the supervisor for a `ResearchPlan`. `researchStep` fans out one task per planned participant plus a topic task, gathers the typed results, runs the PII sanitizer, and records each on the entity. `composeStep` calls the composer and records the finished briefing. Each entity event flows to the view and out over SSE, so the UI watches the briefing move through its states live.

## State machine

A briefing moves RECEIVED → PLANNING → RESEARCHING → COMPOSING → READY on the happy path. A step error at any phase moves it to FAILED with a recorded reason. READY and FAILED are terminal. The generated `index.html` must carry the Lesson 24 CSS overrides so the state names render white-on-dark and the transition labels are not clipped.

## Entity model

The `BriefingEntity` is the single source of truth for one briefing. It contains the meeting topic, the participant list, the status enum, and the lifecycle content — the research plan, the participant profiles, the topic brief, and the composed document — each optional until its event fires. The `RequestQueue` entity records inbound requests and is the workflow's trigger source.
