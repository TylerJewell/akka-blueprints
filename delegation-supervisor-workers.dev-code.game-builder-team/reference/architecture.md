# Architecture — game-builder-team

The diagrams below are the source the Architecture tab renders. Apply the Lesson 24 mermaid CSS overrides so state-box labels and edge labels stay legible on the dark theme.

## Component graph

The system is a supervisor-and-workers team behind a Workflow. A game idea enters through `GameEndpoint` or the `RequestSimulator`, lands in `BuildRequestQueue`, and the `BuildRequestConsumer` starts one `GameBuildWorkflow` per idea. The workflow drives three agents — `GameDirector` supervises while `GameDesigner` and `CodeWriter` do the work — and records every transition on `GameProjectEntity`, which projects into `GameView` for the UI.

See `PLAN.md` Section 1 for the `flowchart` source.

## Interaction sequence

The primary journey runs design → code → test → assemble. The director delegates the design, the designer returns a `GameSpec`, the director then delegates the code, and the code writer returns a `GameCode`. Before the simulated run tool executes, the before-tool-call guardrail screens the code; if it trips, the build is blocked. Otherwise the test gate runs deterministic checks: a pass leads to delivery, repeated failure settles in `TEST_FAILED`.

See `PLAN.md` Section 2 for the `sequenceDiagram` source.

## State machine

A `GameProject` moves `QUEUED → DESIGNING → CODING → TESTING`, then branches to `DELIVERED`, `TEST_FAILED`, or `BLOCKED`. The `TESTING → CODING` edge is the retry loop, bounded by the attempt budget held in `GameProject.testAttempts`.

See `PLAN.md` Section 3 for the `stateDiagram-v2` source.

## Entity model

`GameProjectEntity` is the event-sourced source of truth; its events project into the `GameView` read model. `BuildRequestQueue` is a separate entity that logs each submission for replay.

See `PLAN.md` Section 4 for the `erDiagram` source.
