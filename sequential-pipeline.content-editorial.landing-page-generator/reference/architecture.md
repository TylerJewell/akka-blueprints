# Architecture — Landing Page Generator

The system is a linear, single-track pipeline. A concept enters through one of two doors — an API submit or the background simulator — and flows through three agents in fixed order, with a brand-safety review gate at the end. The diagrams below are the source rendered on the UI's Architecture tab. The generated `index.html` applies the Lesson 24 CSS so state-diagram labels stay white and edge labels are not clipped.

## Component graph

See `PLAN.md` → Component graph. `ConceptSimulator` (a TimedAction) and `LandingPageEndpoint` both feed `ConceptQueue`. `ConceptConsumer` subscribes to the queue's events and starts one `LandingPageWorkflow` per concept. The workflow calls `CopyAgent`, `StructureAgent`, and `CtaAgent` in sequence and writes each result to `LandingPageEntity`. `LandingPagesView` projects the entity's events into a read model that `LandingPageEndpoint` queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

See `PLAN.md` → Interaction sequence. The primary journey: enqueue concept → consumer starts workflow → copy → structure → cta → review. The review step branches: a passing brand-safety evaluation marks the page `READY`; a failing one marks it `FLAGGED`.

## State machine

See `PLAN.md` → State machine. `LandingPageEntity` moves `DRAFTING_COPY → STRUCTURING → WRITING_CTA → REVIEWING`, then terminally to either `READY` or `FLAGGED`. Each transition is driven by one event.

## Entity model

See `PLAN.md` → Entity model. `ConceptQueue` triggers `LandingPage` instances; each `LandingPage` projects one row into `PAGES_VIEW`. The view holds the full row record so the UI can render copy and sections without a second fetch.

## Why these primitives

- **Workflow** for the pipeline because the three stages are durable, ordered, and each calls an agent with its own timeout and retry policy.
- **AutonomousAgent ×3** because each stage runs a single bounded task and returns a typed result.
- **EventSourcedEntity** for the page so every lifecycle transition is an event the view can project and the UI can replay.
- **View + SSE** so the UI reflects state changes without polling.
- **TimedAction + Consumer + queue entity** to model the inbound concept stream with first-party primitives — the runs-out-of-the-box integration form.
