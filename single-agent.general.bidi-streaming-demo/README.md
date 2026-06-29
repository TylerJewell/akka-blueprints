# Akka Sample: Bidi Streaming Demo

A single agent listens on a persistent bidirectional channel, receives a stream of user messages, and replies with a stream of structured response frames — each frame carrying a turn id, a content fragment, and a done flag. The channel stays open until the client closes it or the agent emits a final frame.

Demonstrates the **single-agent** coordination pattern wired for full-duplex communication: an inbound Consumer feeds messages into a channel entity; the agent reads each message and streams response fragments back through a Server-Sent Events endpoint; a before-agent-response guardrail validates every outbound frame before it reaches the wire.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The channel entity, agent, and SSE stream run in-process.

## Generate the system

```sh
cp -r ./single-agent.general.bidi-streaming-demo  ~/my-projects/bidi-demo
cd ~/my-projects/bidi-demo
```

(Optional) Edit `SPEC.md` to change the channel's turn budget, adjust the response-frame schema, or replace the seed conversation scripts.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChannelAgent** — an AutonomousAgent that receives a user message plus channel history as a task attachment and returns a `ResponseFrame` stream — one frame per sentence, ending with a frame whose `done` field is `true`.
- **ChannelWorkflow** — orchestrates receive-wait → respond → flush per inbound message.
- **ChannelEntity** — an EventSourcedEntity holding per-channel state: open messages, response frames, and lifecycle.
- **MessageForwarder** — a Consumer that subscribes to `MessageReceived` events and starts a workflow run per message.
- **ChannelView + ChannelEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the turn budget or idle-timeout for a channel (`channelTtlSeconds`).
- `SPEC.md §5` — extend `ResponseFrame` with domain-specific metadata (e.g., `sourceRef`, `confidence`).
- `prompts/channel-agent.md` — narrow the agent's persona or constrain its reply format (e.g., restrict to JSON-only frames for a machine-readable bidi protocol).
- `eval-matrix.yaml` — there are no controls for this baseline (tier = baseline, domain = general); after customising for your use case, add controls here and reference them in SPEC.md §8.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user opens a channel, sends a message, and receives a streamed reply broken into frames — each frame visible in the UI as it arrives.
2. The agent returns a malformed frame on one iteration — the `before-agent-response` guardrail rejects it; the agent retries; the correct frame streams to the client.
3. A channel that exceeds its turn budget receives a `CLOSED` status; no further messages are accepted.
4. A second client subscribing to the same channel's SSE stream sees all frames emitted so far, then continues receiving new frames in real time.

## License

Apache 2.0.
