# Architecture

Narrative around the four diagrams in `PLAN.md`. The generated system renders the same mermaid sources on the Architecture tab, with the Lesson 24 CSS overrides so state-box labels and edge labels stay legible.

## Component graph

A request enters through `GreetingEndpoint` (`POST /api/greetings`) or is dripped by `RequestSimulator` every 60 seconds. Either path writes a `GreetingSubmitted` event onto `RequestQueue`. `GreetingRequestConsumer` subscribes to those events and starts one `GreetingWorkflow` per submission, keyed by `requestId`.

The workflow is the supervisor. It asks `GreetingCoordinator` to decompose the submission into work specs, fans the work out to `GreetingWriter` and `ActionAdvisor` in parallel, then asks `GreetingCoordinator` to compose the final response. Every state transition is written as a command to `GreetingEntity`, whose events project into `GreetingView`. The endpoint reads and streams the view.

## Interaction sequence

The sequence diagram traces the primary journey: decompose, parallel fan-out to both workers, join, compose, persist as COMPLETED. The `par` block is the delegation-supervisor-workers core — both workers run concurrently and the workflow joins their results before handing off to the coordinator's COMPOSE task.

## State machine

`GreetingRequest` moves `PENDING → IN_PROGRESS`, then to one of two terminals: `COMPLETED` (compose succeeded) or `DEGRADED` (a worker timed out). Because this baseline carries no guardrail, there is no BLOCKED terminal. `COMPLETED` and `DEGRADED` are both final — no further transitions occur.

## Entity model

`RequestQueue` seeds one `GreetingRequest` per submission. A request owns at most one `DraftMessage`, one `FollowUpAction`, and one `GreetingResponse`. The view row mirrors the request with `Optional<T>` on every field that is null before its transition (Lesson 6).
