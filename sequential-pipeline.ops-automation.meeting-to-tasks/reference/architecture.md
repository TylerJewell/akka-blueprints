# Architecture — meeting-to-tasks

Narrative around the four diagrams in `PLAN.md`. The generated UI renders the same diagrams on the Architecture tab.

## Component graph

The pipeline is a single `Workflow` (`MeetingPipelineWorkflow`) that owns the ordered steps. Two entry points feed it: `MeetingEndpoint` (a user POSTs notes) and `NotesSimulator` (a `TimedAction` that drips canned notes every 45 seconds). The workflow calls plain Java helpers for the side-effecting work — `PiiSanitizer` to redact, `TaskExtractionAgent` to extract, `TrelloClient` / `SlackClient` / `CsvWriter` for the outputs — and records each transition on `MeetingEntity`. `MeetingsView` projects the entity's events into a read model that `MeetingEndpoint` serves as a list and an SSE stream.

## Interaction sequence

The primary journey: a user submits notes, the workflow sanitizes and extracts, then pauses at `AWAITING_APPROVAL`. While paused it self-schedules a 5-second resume poll against `MeetingEntity`. When a human POSTs approve, the poll observes the `APPROVED` status, runs the before-tool-call guardrail, and performs the board push, CSV export, and Slack notification in order before recording completion. The UI sees each transition over SSE.

## State machine

A meeting moves `RECEIVED → SANITIZED → EXTRACTED → AWAITING_APPROVAL`, then forks to `APPROVED → COMPLETED` or to `REJECTED`. Any step can move the run to `FAILED` through the workflow's recovery path. `COMPLETED`, `REJECTED`, and `FAILED` are terminal. Every nullable lifecycle field on the `Meeting` row record is `Optional<T>` so the view materializer accepts partially-filled rows (Lesson 6).

## Entity model

`MeetingEntity` is event-sourced. Its events (`MeetingReceived`, `NotesSanitized`, `TasksExtracted`, `MeetingApproved`, `MeetingRejected`, `TasksPushedToBoard`, `CsvExported`, `SlackNotified`, `MeetingCompleted`, `MeetingFailed`) each apply to the `Meeting` state and project into one `meetings_view` row. The view exposes a single query, `getAllMeetings`; status filtering happens client-side in the endpoint because Akka cannot auto-index the enum column (Lesson 2).
