# Architecture — meeting-assistant-zoom

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one agent that runs three tasks in sequence. `MeetingEndpoint` accepts a `{meetingTitle, rawTranscript}` POST, writes `MeetingCreated` onto `MeetingEntity`, and starts `MeetingPipelineWorkflow` keyed by `"pipeline-" + meetingId`. Before the first agent call, `PiiSanitizer.sanitize(rawTranscript)` runs synchronously — no LLM involved — and the sanitized text plus a `RedactionAudit` are recorded on the entity with `TranscribeStarted`. The workflow's first step (`transcribeStep`) then calls `MeetingAgent` with `TaskDef.taskType(TRANSCRIBE_MEETING)` and a `phase = TRANSCRIBE` metadata tag, passing only the sanitized text. The agent invokes `TranscribeTools.segmentTranscript` and `TranscribeTools.labelSpeakers`; every call passes through `ScopeGuardrail` first. Once the agent returns a `Transcript`, the workflow writes `TranscriptProduced` onto the entity and advances to `summarizeStep` — same pattern, the SUMMARIZE task carries `phase = SUMMARIZE`. Then `dispatchStep` runs with `phase = DISPATCH`. After `FollowUpsDispatched` lands, `evalStep` runs `ActionItemScorer` over the recorded `(MeetingPackage, MeetingSummary, Transcript)` triple — no LLM call — and writes `EvaluationScored`. `MeetingView` projects every event into a read-model row; `MeetingEndpoint` serves the read model to the UI over REST and SSE.

The graph has exactly one LLM-calling component. `PiiSanitizer` and `ActionItemScorer` are both deterministic and rule-based; that is what makes this a faithful **single-agent sequential-pipeline** example with defence-in-depth at both ends.

## Interaction sequence

The sequence traces the happy path (J1). Three properties are worth pausing on:

1. The PII sanitizer runs before any LLM call. `PiiSanitizer.sanitize` is invoked inside `transcribeStep`, before `runSingleTask` is called. The agent's `TaskDef.instructions` carry only the sanitized text; the raw transcript never enters any model context.
2. The task boundary IS the dependency contract. Between `transcribeStep` and `summarizeStep`, the workflow writes `TranscriptProduced` onto the entity. The next step then reads `transcript` from the entity to build the SUMMARIZE task's instruction context. The agent never sees transcribe-phase context inside the summarize task's conversation; the typed handoff is the only path information travels.
3. Every tool call is filtered through `ScopeGuardrail`. The guardrail reads the in-flight task's `phase` metadata and the current `MeetingEntity.status`, and applies the per-phase accept matrix. For DISPATCH-phase tools, it additionally checks that every attendee and assignee pseudonym is in the recorded participant list. A misordered or out-of-scope call is rejected before the tool body executes.

The agent calls themselves are bounded by per-step timeouts (90 s on transcribe / summarize / dispatch). `evalStep` is synchronous and finishes in milliseconds.

## State machine

Nine states. The interesting paths:

- The happy path walks `CREATED → TRANSCRIBING → TRANSCRIBED → SUMMARIZING → SUMMARIZED → DISPATCHING → DISPATCHED → EVALUATED`.
- Three failure transitions land in `FAILED`: an agent error during `TRANSCRIBING`, `SUMMARIZING`, or `DISPATCHING`. A `FAILED` meeting's prior data is preserved on the entity — the UI shows the partial state.
- `ScopeRejected` is a side-event recorded for audit; it does not transition status. Only an exhausted retry budget or a step timeout transitions to `FAILED`.

There is no `APPROVED` or `SENT` state. The meeting package is advisory; the human reviewer confirms each follow-up before it leaves the system. The blueprint deliberately stops at `EVALUATED`.

## Entity model

`MeetingEntity` is the source of truth. It emits ten event types — three lifecycle starts, three lifecycle completions, the evaluation, the scope-rejection audit, the failure, and the initial creation. `MeetingView` projects every event into a row used by the UI. `MeetingPipelineWorkflow` both reads (`getMeeting`) and writes (`startTranscribe`, `recordTranscript`, `startSummarize`, `recordSummary`, `startDispatch`, `recordPackage`, `recordEvaluation`, `recordScopeRejection`, `fail`) on the entity. The relationship between `MeetingAgent` and each typed result is "returns" — the agent's task result becomes the corresponding event payload.

## Defence-in-depth governance flow

For any meeting that lands in the entity log, the transcript passed through:

1. **PII sanitizer** — runs synchronously before the first LLM call. Participant names, emails, and phone numbers are replaced with stable pseudonyms. The raw transcript is stored encrypted on the entity and never forwarded to the model.
2. **Phase-order + write-scope guardrail** — every tool call is filtered. A DISPATCH-phase tool called during TRANSCRIBE is rejected before the tool body runs; a `ScopeRejected` event records the violation for audit. A dispatch write for an unrecognized pseudonym is rejected before any calendar or task record is created.
3. **MeetingAgent (3 task runs)** — three model calls, three structured outputs. Each task's typed result is the dependency handoff to the next phase.
4. **On-decision evaluator** — every emitted `MeetingPackage` gets a 1–5 action-item completeness score. Action-item coverage, assignee traceability, event-attendee traceability, and item parity are each worth one point on a base of 1.

Each step is independent. The PII sanitizer does not check phase order; the guardrail does not check completeness; the evaluator does not check PII. Removing any one of them opens an explicit gap the others do not silently cover.
