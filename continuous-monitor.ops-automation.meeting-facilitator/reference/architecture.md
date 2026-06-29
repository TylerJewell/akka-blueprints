# Architecture — meeting-facilitator

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`MeetingSessionPoller` is the heartbeat — a TimedAction that ticks every 20 s and writes simulated `SegmentReceived` events into `MeetingSessionQueue` (event-sourced for audit). A `TranscriptSanitizer` Consumer subscribes to that queue, redacts speaker names and other PII from the transcript and chat thread, and emits `SegmentSanitized` on the per-session `MeetingSessionEntity`. That sanitized event starts a `MeetingFacilitatorWorkflow` instance, which orchestrates: generate notes → generate chat recap → guardrail check → publish or suppress.

`EvalRunner` runs alongside as a second TimedAction, ticking every 30 minutes and scoring sampled published sessions.

## Interaction sequence

The sequence traces the happy path. Note the two AI calls (`NotesSummaryAgent` and `ChatSummaryAgent`) run sequentially inside the workflow before the guardrail step evaluates their combined output. The guardrail is the only gate between summarization and publication — there is no human-in-the-loop step for this baseline (the `risk-survey.yaml` declares `human_on_loop: true`, meaning a reviewer can audit but is not required to approve every publication).

## State machine

Five states. The interesting branch is at SUMMARIZED: if `guardrailStep` produces a passing `GuardrailVerdict`, the session moves to PUBLISHED; if the guardrail fires, the session moves to SUPPRESSED. Both are terminal. There is no retry path — a suppressed session stays suppressed and surfaces in the UI for human review.

## Entity model

`MeetingSessionEntity` is the source of truth; it emits nine distinct event types covering the full lifecycle including eval. `MeetingSessionQueue` is the upstream audit log — only `TranscriptSanitizer` subscribes to it. The raw `TranscriptSegment` (with unredacted speaker names and text) is retained in the entity's `rawSegment` field for audit purposes but is never projected into `MeetingSessionView`.

## Defence-in-depth governance flow

For any summary that reaches PUBLISHED, the session passed through:
1. **PII sanitizer** — the model never saw raw speaker names, emails, or phone numbers.
2. **NotesSummaryAgent** — typed classifier that cannot stray outside its `MeetingNotes` output schema.
3. **ChatSummaryAgent** — same schema constraint on chat output.
4. **Before-agent-response guardrail** — independent policy check on both outputs before they are persisted as published.

Each step is independent. A guardrail failure does not silently degrade to a warning — it changes the terminal state from PUBLISHED to SUPPRESSED and records the `reason` on the entity.
