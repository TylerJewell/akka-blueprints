# Architecture

Narrative around the four diagrams in `PLAN.md`. The system is a repeating 3-step graph in the human-in-loop-gate pattern: draft chapter, screen draft, await reader direction.

## Component graph

`StoryEndpoint` accepts a premise and starts `StoryWorkflow` with a fresh story id. Each workflow turn drives `StoryDrafterAgent` to write a chapter, then drives `ContentGuardAgent` to screen it. If the screen passes, the chapter is written to `StoryEntity` and the workflow pauses awaiting direction. The reader calls continue (with direction) or end through the endpoint, which transitions `StoryEntity`. On continue, the workflow reads the updated status and loops back to draft the next chapter. On end, the workflow exits. `StoryEntity` events project into `StoriesView`, which the endpoint queries and streams over SSE. `AppEndpoint` serves the static UI.

## Interaction sequence

The primary journey runs premise → draft → screen → pause → reader direction → draft → screen → pause → ... → reader ends. The await-direction step does not hold a thread: `awaitDirectionStep` reads `StoryEntity.getStory`, and while the status is `AWAITING_DIRECTION` it self-schedules a 5-second resume timer. When the reader calls continue, the entity transitions to `DRAFTING`; the next poll moves the workflow into `draftChapterStep`. A `COMPLETED` or `SCREENED_OUT` status on any poll terminates the workflow immediately.

## State machine

A story moves `AWAITING_DIRECTION → DRAFTING → AWAITING_DIRECTION` on each successful turn. The reader ends the story from `AWAITING_DIRECTION`, transitioning to terminal `COMPLETED`. If the content guard rejects a draft, the story transitions to terminal `SCREENED_OUT` from `DRAFTING` without writing a chapter. Both terminal states prevent any further workflow activity.

## Entity model

`StoryEntity` is event-sourced; each command emits one event that the applier folds into the `Story` state. `Story` carries a list of `Chapter` records that grows by one per successful turn. `StoriesView` projects the same events into a row keyed by story id. There is one view query, `getAllStories`; callers filter by status client-side because Akka cannot auto-index the enum column (Lesson 2).
