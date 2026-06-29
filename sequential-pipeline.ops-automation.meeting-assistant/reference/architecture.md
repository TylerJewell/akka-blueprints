# Architecture

Meeting Assistant is a sequential pipeline. One meeting moves through three ordered stages, each gated on the previous one finishing: redact personal data, extract action items, dispatch to Trello and Slack. The four diagrams below are reproduced from `PLAN.md`; this file gives the narrative around each.

## Component graph

See the `flowchart TB` in `PLAN.md`. Work enters two ways: a user POST to `MeetingEndpoint`, or `NotesSimulator` dripping a canned line every 30 seconds. Both land in `InboundNotesQueue`, whose `InboundNotesQueued` event wakes `NotesConsumer`, which starts one `PipelineWorkflow` per submission. The workflow drives `MeetingEntity` through its lifecycle and calls `ActionExtractorAgent` for the one model-backed stage. `MeetingsView` projects entity events for the UI to list and stream. `DispatchRetryMonitor` re-drives meetings that reached `EXTRACTED` but never dispatched.

## Interaction sequence

See the `sequenceDiagram` in `PLAN.md`. The two `Note over` blocks mark the governance points: redaction runs before any model call, and the before-tool-call guardrail runs before each external write. The agent only ever receives redacted text.

## State machine

See the `stateDiagram-v2` in `PLAN.md`. A meeting advances `RECEIVED → SANITIZED → EXTRACTED → DISPATCHED`. The only branch is at dispatch: a guardrail refusal sends it to `FAILED` instead. Both `DISPATCHED` and `FAILED` are terminal. State and transition labels use the Lesson 24 CSS overrides so they render white on the dark theme.

## Entity model

See the `erDiagram` in `PLAN.md`. `MeetingEntity` holds one `Meeting` with an embedded list of `ActionItem`. `InboundNotesQueue` holds the raw submission until a workflow starts. The redacted notes live on the meeting; the raw notes are not persisted past the queue event.
