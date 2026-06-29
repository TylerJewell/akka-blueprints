# Akka Sample: Per-Customer Success CRM Agents

A control-plane agent manages a fleet of per-customer success agents. When a new customer is registered, the control plane spawns a dedicated `CustomerAgent` for that customer — it researches their background, dispatches a personalized welcome email through an outbound email tool, maintains a durable memory of every interaction, and then stays resident to support the customer through an ongoing chat. The fleet coordinator monitors all agents at runtime, and a periodic eval measures fleet-wide quality.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. The customer registry, the agent fleet, the memory store, and the chat channel are all modelled inside the same service with first-party Akka primitives.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./composite-multi-team.cx-support.akka-per-tenant-agent-fleet  ~/my-projects/per-tenant-agent-fleet
cd ~/my-projects/per-tenant-agent-fleet
```

(Optional) Edit `SPEC.md` to change the customer simulator cadence, the outbound email tool's guardrail policy, or the fleet monitoring thresholds.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FleetCoordinator** — AutonomousAgent that acts as the control plane: on a new customer it spawns the `CustomerAgent` instance, and under a periodic eval it samples active agents for quality scoring.
- **CustomerAgent** — the per-customer agent. Receives a customer record, runs `BackgroundResearcher` to build a profile, calls the outbound email tool (gated by a before-tool-call guardrail) to send a welcome, persists a memory, then enters an ongoing chat loop answering the customer's questions.
- **BackgroundResearcher** — AutonomousAgent that produces a `CustomerProfile` from structured CRM data. Run as one instance per customer.
- **ChatResponder** — AutonomousAgent that handles each incoming chat message for a given customer, drawing on the persisted memory.
- **AgentFleetWorkflow** — the per-customer durable pipeline: register → research → welcome → ready → ongoing-chat, with a PII sanitizer pass on every input.
- **CustomerRegistry** + **AgentMemoryEntity** — the customer master record and the per-customer interaction memory.
- **FleetStatusView** + **ChatHistoryView** — the fleet overview and the per-customer conversation read models the UI streams.
- **CustomerIngestConsumer** — listens to `CustomerRegistry` events and starts one `AgentFleetWorkflow` per new customer.
- **FleetEvalConsumer** — fires a non-blocking periodic quality eval on welcome and chat stages.
- **CustomerSimulator** — drips sample customer records every 60 s so the board is never empty when first loaded.
- **StaleAgentMonitor** — reactivates dormant agents that have received no interaction for a configurable window.
- **FleetEndpoint** + **AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the customer topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §11` — adjust the fleet size cap or the ongoing-chat session window.
- `prompts/background-researcher.md`, `prompts/chat-responder.md` — tune each agent's domain expertise or tone.
- `eval-matrix.yaml` — extend the before-tool-call guardrail's refused-operation list, or change the fleet-eval thresholds.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Register a customer → the control plane spawns a `CustomerAgent` → the background researcher builds a profile → the welcome email is dispatched under the guardrail → the agent enters ready state and answers a chat message.
2. Two simultaneous registrations → each customer gets exactly one dedicated agent; no merging.
3. The outbound email tool is called with a disallowed recipient or oversized body → the before-tool-call guardrail blocks it; no email is sent and the reason is recorded.
4. A PII field (e.g., raw SSN) appears in a chat input → the sanitizer strips it before the agent sees it.
5. After a customer's agent has been running for the monitoring window, the deployer-runtime monitor surfaces a fleet health summary in the UI.

## License

Apache 2.0.
