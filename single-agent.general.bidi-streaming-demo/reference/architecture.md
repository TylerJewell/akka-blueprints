# Architecture — bidi-demo

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one conversational LLM call per channel turn. `ChannelEndpoint` accepts two kinds of requests: open a channel and send a message. Opening writes a `ChannelOpened` event; sending writes a `MessageReceived` event to `ChannelEntity`. The `MessageForwarder` Consumer subscribes, mints a `turnId`, calls `ChannelEntity.startTurn`, and starts a `ChannelWorkflow` instance for that turn. The workflow's `respondStep` calls `ChannelAgent` — the single AutonomousAgent — with the new message as `TaskDef.instructions(...)` and the channel's prior turn history as a `TaskDef.attachment("history.json", ...)`. The agent's `before-agent-response` guardrail (`FrameGuardrail`) validates each candidate `ResponseFrameList`. Once a frame list passes, `respondStep` iterates it and calls `ChannelEntity.publishFrame` for each frame — each `FramePublished` event propagates immediately to the SSE stream, giving clients a frame-by-frame streaming experience. `flushStep` checks the turn budget and closes the channel if exhausted. `ChannelView` projects every entity event into a read-model row; `ChannelEndpoint` serves the read model and the SSE stream to the UI.

The graph has no second agent. The turn-budget checker in `flushStep` is deterministic — not an LLM call. That is what makes this blueprint a faithful **single-agent** example.

## Interaction sequence

The sequence traces the happy path (J1). Three distinct pacing moments:

1. The `MessageForwarder` subscription lag between `MessageReceived` and the workflow start — sub-second in normal operation.
2. The `awaitTurnStep` polling loop inside the workflow — polls `ChannelEntity` every 500 ms up to its 10 s timeout, advancing as soon as the turn is in `PROCESSING` status.
3. The agent call, bounded by `respondStep`'s 60 s timeout. The frame-publishing loop runs inside the same step after the agent returns; each `publishFrame` call is a synchronous entity command, completing in milliseconds.

## State machine

Four channel states. The interesting paths:

- The happy path opens at `OPEN`, transitions to `ACTIVE` on the first message, stays `ACTIVE` across subsequent turns, and terminates at `CLOSED` when the turn budget is reached.
- Two paths land in `FAILED`: an error before the first message (`OPEN → FAILED`) or an unrecoverable error mid-conversation (`ACTIVE → FAILED`). Prior turn data is preserved on the entity.
- There is no `ARCHIVED` or `EXPIRED` state. The channel closes when the budget is exhausted; a new channel is opened for further conversation.

## Entity model

`ChannelEntity` is the source of truth. It emits seven event types. `ChannelView` projects every event into a row used by the UI. `MessageForwarder` subscribes to entity events to start workflow runs. `ChannelWorkflow` both reads (`getChannel`) and writes (`startTurn`, `publishFrame`, `completeTurn`, `failTurn`, `closeChannel`) on the entity. `ChannelAgent` returns a `ResponseFrameList`; the workflow unpacks it into individual `publishFrame` calls.

## Bidirectional streaming flow

For any frame that appears in the SSE stream, the message passed through:

1. **MessageReceived** — the raw user message lands in the entity; the channel history is preserved for audit.
2. **ChannelAgent** — one model call per turn, returning a list of structured frames; the full conversation history is attached as context.
3. **before-agent-response guardrail** — malformed frame lists (empty content, wrong turnId, index gaps) are caught before any frame reaches the entity or the SSE wire.
4. **Frame-by-frame publishing** — each passing frame is written as a `FramePublished` event and streamed to all subscribers of that channel's SSE endpoint immediately.

Each step is independent. The guardrail does not inspect history; the publisher does not re-validate frames. Removing the guardrail would allow malformed frames to reach clients with no other check intercepting them.
