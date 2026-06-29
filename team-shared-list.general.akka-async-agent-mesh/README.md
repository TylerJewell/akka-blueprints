# Akka Sample: Async Multi-Agent Communication

Two independent agents, each maintaining their own long-term memory, exchange fire-and-forget messages through a `send_message_to_agent_async` tool. Neither agent blocks waiting for a reply — each continues its own work while the receiving agent processes the incoming message on its own schedule. Demonstrates the **team-shared-list** coordination pattern applied to async inter-agent messaging with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The agent roster, message bus, memory store, and delivery tracking are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./team-shared-list.general.akka-async-agent-mesh  ~/my-projects/akka-async-agent-mesh
cd ~/my-projects/akka-async-agent-mesh
```

(Optional) Edit `SPEC.md` to change the agent roster, the initial message scenarios the simulator injects, or the model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **DispatcherAgent** — AutonomousAgent that composes outbound messages and calls the `send_message_to_agent_async` tool. Fire-and-forget: it receives a delivery receipt and moves on without waiting for a response.
- **ProcessorAgent** — AutonomousAgent that receives inbound messages from other agents, processes them against its own memory, and may enqueue follow-up messages back to the dispatcher.
- **MessageDispatchWorkflow** — Workflow that runs the dispatcher, records the outbound message on the shared list, and stores the delivery receipt.
- **MessageProcessingWorkflow** — Workflow that runs the processor for each inbound message, writes the processing result to the shared list, and checks the PII sanitizer output.
- **MessageEntity** — one EventSourcedEntity per in-flight message; tracks delivery and processing lifecycle.
- **AgentMemoryEntity** — one EventSourcedEntity per agent instance; holds the agent's durable long-term memory entries.
- **DeliveryReceiptEntity** — EventSourcedEntity recording confirmed receipt acknowledgements.
- **MessageBusView** — the shared message list the UI and the agents read; one row per message across both directions.
- **MessageBusConsumer** — Consumer that subscribes to outbound message events and starts a `MessageProcessingWorkflow` for each.
- **ScenarioSimulator** — TimedAction that drips a sample dispatch scenario every 60 seconds.
- **StaleMessageMonitor** — TimedAction that flags messages neither delivered nor processed within a timeout window.
- **MeshEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the scenario scripts the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — change the agent roster (`agent-alpha`, `agent-beta`) to a different pair.
- `prompts/dispatcher.md` — narrow the dispatcher to a specific message domain.
- `prompts/processor.md` — tune the processor's memory-recall and PII-filtering behavior.
- `eval-matrix.yaml` — extend the before-tool-call guardrail's disallowed-recipient list if you wire additional agent identities.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Trigger a dispatch → dispatcher composes and sends a message → processor receives and processes it → message reaches `PROCESSED` on the board.
2. Dispatcher fires a message and continues immediately without waiting; the processor's reply arrives asynchronously; both agents progress in parallel.
3. A message carrying PII triggers the sanitizer; the sanitized payload is stored; the original value is never written to any entity or view.
4. Dispatcher attempts to address a message to an unknown agent id; the before-tool-call guardrail blocks the send before it executes.
5. The operator halts the mesh; no new messages are dispatched and no processing begins; resume restores normal flow.

## License

Apache 2.0.
