# Architecture — voice-agent

The generated system's Architecture tab renders the four mermaid diagrams from `PLAN.md`. Narrative below.

## Component graph

The graph is centred on one decision-making LLM call. `ConversationEndpoint` accepts a new session or a new turn and writes an `AudioReceived` event onto `ConversationEntity`. The `TranscriptSanitizer` Consumer subscribes, redacts PII from the raw transcript, and writes the sanitized form back via `attachSanitizedTranscript`. The same Consumer then starts a `ConversationWorkflow` instance for that turn. The workflow's `respondStep` calls `VoiceConversationAgent` — the single AutonomousAgent — with the conversation history as `TaskDef.instructions(...)` and the sanitized transcript as a `TaskDef.attachment(...)`. The agent's `before-agent-response` guardrail (`ReplyGuardrail`) validates each candidate response before audio synthesis fires. Once a reply passes, the workflow calls `AudioSynthesiser` in `synthesiseStep`, writes `AudioSynthesised`, and the turn is complete. `ConversationView` projects every entity event into a read-model row; `ConversationEndpoint` serves the read model to the UI over REST and SSE.

The graph has no second agent. `AudioSynthesiser` is a deterministic stub class that converts reply text to bytes without any model call. That is what makes this blueprint a faithful **single-agent** example: exactly one component talks to a model.

## Interaction sequence

The sequence traces the happy path (J1). Two moments where the system waits on something:

1. The `TranscriptSanitizer` subscription lag between `AudioReceived` and `TranscriptSanitized` — sub-second in normal operation.
2. The `awaitSanitizedStep` polling loop inside the workflow — polls `ConversationEntity` every 1 s up to its 15 s timeout, advancing as soon as the current turn's `sanitized().isPresent()` returns true.

The agent call itself is bounded by `respondStep`'s 60 s timeout. `synthesiseStep` is synchronous and finishes in milliseconds for the in-process stub — no external service, no LLM call.

## State machine

Six states per turn. The interesting paths:

- The happy path is `AUDIO_RECEIVED → TRANSCRIPT_SANITIZED → RESPONDING → REPLY_RECORDED → AUDIO_SYNTHESISED`.
- Two failure transitions land in `FAILED`: a sanitizer error during `AUDIO_RECEIVED`, and an agent error (or guardrail-exhaustion) during `RESPONDING`. A `FAILED` turn's prior data is preserved on the entity — the UI shows the partial state for the supervisor.
- There is no `CLOSED` per-turn state. The session (`ConversationSession`) has its own `SessionStatus` (`ACTIVE`, `CLOSED`, `FAILED`) managed by `SessionClosed` events at the session level, not the turn level.

## Entity model

`ConversationEntity` is the source of truth. It emits seven event types. `ConversationView` projects every event into a row used by the UI. `TranscriptSanitizer` subscribes to entity events to compute the sanitized transcript. `ConversationWorkflow` both reads (`getSession`) and writes (`markResponding`, `recordReply`, `recordAudio`, `failTurn`) on the entity. The relationship between `VoiceConversationAgent` and `AgentReply` is "returns" — the agent's task result is the reply record. `AudioSynthesiser` is called by the workflow, not by the entity; the entity only records the resulting `AudioResponse` when the workflow writes it back.

## Defence-in-depth governance flow

For any reply that reaches the caller, the transcript passed through:

1. **PII sanitizer** — the model never sees caller names, account numbers, or payment details; the audit log retains the raw transcript.
2. **VoiceConversationAgent** — one model call, one structured output per turn.
3. **before-agent-response guardrail** — over-length replies, disallowed phrases, and out-of-enum tones are caught before the response leaves the agent loop and before `AudioSynthesiser` is called.

Each step is independent. Removing one of them opens an explicit gap the other does not silently cover.
