# Architecture — morning-email-debrief

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

`MailboxPoller` is the heartbeat — a TimedAction that ticks every 60 s and writes simulated `EmailReceived` events into `EmailQueue` (event-sourced for audit), grouping each batch under a generated `runId`. An `EmailPiiSanitizer` Consumer subscribes to that queue, redacts each email's subject and body, and emits `EmailSanitized` on the per-email `EmailEntity`. It also creates or increments the corresponding `DebriefEntity` run. A `DebriefWorkflow` instance per run then orchestrates: wait for all emails in the batch to be sanitized → call `EmailSummarizerAgent` once per email → call `DebriefAssemblerAgent` once to combine all entries → emit `DebriefReady`.

`EvalRunner` runs alongside as a second TimedAction, ticking every 60 minutes and scoring sampled READY debriefs.

## Interaction sequence

The sequence traces the happy path (J1). Note three distinct phases:

1. The `MailboxPoller` drips N emails into the queue; `EmailPiiSanitizer` processes each one concurrently (Consumer fan-out) — sub-second per email.
2. `DebriefWorkflow` polls `EmailView` in `awaitAllSanitizedStep` until all emails for the run are SUMMARISED (poll every 10 s, max 10 minutes). This is the pipeline's natural back-pressure gate.
3. Assembly is a single `DebriefAssemblerAgent` call that receives all entries and returns one `MorningDebrief`.

## State machine

Four states for a debrief run. FAILED is a terminal state triggered if assembly times out or the assembler returns an error. READY transitions to itself when `DebriefEvalScored` is applied — the eval does not change the run's primary status, it augments it.

The email-level state machine (`EmailEntity`) is a straight three-state sequence: RECEIVED → SANITIZED → SUMMARISED. No branching.

## Entity model

`EmailEntity` is the source of truth per email; it emits three event types. `DebriefEntity` is the source of truth per run; it emits five event types including the eval. Both are projected into Views that `DebriefEndpoint` queries and streams via SSE.

`EmailQueue` is the upstream audit log — only `EmailPiiSanitizer` subscribes to it, and it holds the raw pre-sanitization payloads that the application's own Views never expose.

## Governance flow

For any debrief that reaches READY, every email in the batch passed through:

1. **PII sanitizer** — `EmailPiiSanitizer` redacted all identifiers before any LLM prompt was constructed.
2. **EmailSummarizerAgent** — typed classifier, receives only `SanitizedEmailPayload`.
3. **DebriefAssemblerAgent** — receives only `DebriefEntry` values (summaries), never raw email content.
4. **Eval-periodic** — `EvalRunner` scores coverage and tone on the assembled output; score degradation surfaces automatically.

No step receives raw email content after the Consumer runs. The audit log (`EmailQueue` + `EmailEntity.incoming`) is the only place raw content is retained, and it is never served through the public API.
