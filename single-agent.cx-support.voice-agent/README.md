# Akka Sample: Voice Agent

A single voice-conversation agent accepts spoken audio from a caller, transcribes it with an STT service, generates a response, and synthesises that response into audio with a TTS service. The transcript rides into the agent as a task attachment, never as inline prompt text.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a PII sanitizer that runs on every inbound transcript before the agent ever sees it, and a `before-agent-response` guardrail that validates every spoken response before it is converted to audio and played back to the caller.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. STT and TTS are simulated in-process; the agent's "tool calls" are local stubs. The blueprint runs out of the box.

## Generate the system

```sh
cp -r ./single-agent.cx-support.voice-agent  ~/my-projects/voice-agent
cd ~/my-projects/voice-agent
```

(Optional) Edit `SPEC.md` to point at a different persona or change the seeded conversation scenarios (e.g., switch from general customer support to a billing or technical-support persona).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **VoiceConversationAgent** — an AutonomousAgent that accepts a transcript as a task attachment and returns a typed `AgentReply` carrying the text response plus a suggested voice tone.
- **ConversationWorkflow** — orchestrates transcribe-wait → respond → synthesise per inbound audio chunk.
- **ConversationEntity** — an EventSourcedEntity holding the per-session conversation lifecycle.
- **TranscriptSanitizer** — a Consumer that subscribes to `AudioReceived` events, redacts PII from the raw transcript, and emits `TranscriptSanitized` back to the entity.
- **ConversationView + ConversationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded conversation scenarios for your own (the JSONL file under `src/main/resources/sample-events/scenarios.jsonl` after generation).
- `SPEC.md §5` — extend `AgentReply` with additional fields (e.g., `escalationFlag`, `topicClassification`, `confidenceScore`).
- `prompts/voice-conversation.md` — narrow the agent's role and persona (a telecoms deployer might constrain it to a billing-support persona; a healthcare deployer to appointment scheduling with HIPAA constraints).
- `eval-matrix.yaml` — wire a real STT provider (e.g., a Whisper-backed transcription service) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A caller sends audio → it is transcribed → the transcript is sanitized → the agent replies → the response is synthesised and returned to the UI.
2. The agent produces a response containing a disallowed phrase on its first try → the `before-agent-response` guardrail rejects it → the agent retries → a clean response plays.
3. A transcript containing a caller's name and phone number is submitted; the LLM call log shows only the redacted forms; the entity's `rawTranscript` retains the originals for audit.
4. A multi-turn session accumulates all turns in the correct order on the entity and the UI displays the full conversation history.

## License

Apache 2.0.
