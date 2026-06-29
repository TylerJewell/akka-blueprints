# Architecture

The system is a sequential pipeline: a workflow drives a fixed order of agent steps, each consuming the prior step's typed output, with one before-agent-response guardrail wired into the path. Everything runs in one process, so the sample runs out of the box. The four diagrams below are the source the Architecture tab renders.

## Component graph

Every component, colored by Akka primitive. Solid arrows are synchronous commands, dashed arrows are event subscriptions, dotted arrows are scheduled ticks. See the `flowchart TB` block in `PLAN.md`.

`BriefSimulator` drips canned briefs into `BriefQueue`; `PostEndpoint` also enqueues briefs from real submissions. `BriefConsumer` reacts to each `BriefQueued` event and starts one `PostWorkflow`. The workflow calls `CaptionAgent` and then `ImagePromptAgent`, running the brand/safety guardrail on each result and writing into `PostEntity`. `PostsView` projects `PostEntity` events into the read model the UI streams.

## Interaction sequence

The primary journey: submit a brief, the caption agent writes the caption, the guardrail checks it, the image-prompt agent writes the image prompt, the guardrail checks it, and the post is marked ready. The `Note over` blocks mark the two guardrail checks and the terminal ready state. See the `sequenceDiagram` block in `PLAN.md`.

## State machine

`PostEntity` lifecycle: `COMPOSING` → `PROMPTING` → `READY`, with `BLOCKED` (from a failed brand/safety check at either step) and `FAILED` (from exhausted workflow retries) as terminal branches. See the `stateDiagram-v2` block in `PLAN.md`. The Architecture tab must apply the Lesson 24 CSS overrides so state names and transition labels render legibly.

## Entity model

`BriefQueue` records inbound briefs and starts `PostEntity` instances; `PostEntity` projects into `PostsView`. See the `erDiagram` block in `PLAN.md`.

## Concurrency

Agent-calling steps carry explicit 60-second timeouts; the finalize step is a local entity write. The workflow id is the post id, so re-delivery of a queued brief reuses the same workflow instance. A failed guardrail check transitions the post to `BLOCKED` with no further step. Full notes in `PLAN.md`.
