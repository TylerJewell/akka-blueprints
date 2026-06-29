# Akka Sample: WhatsApp Fintech Agent

A single financial-query agent receives customer messages from a WhatsApp-channel gateway, resolves account context, and responds with account information or initiates a transaction. Inbound messages are PII-sanitized before the agent sees them. Every tool call that moves money passes through a before-tool-call guardrail. Transactions above a configurable value threshold are paused for a human confirmation step before execution completes.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that runs inside a Consumer before the agent ever sees the message text, a `before-tool-call` guardrail that validates payment instructions before any funds-movement tool fires, and an application-level human-in-the-loop (HITL) gate that requires explicit confirmation for large-value operations.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the WhatsApp channel is simulated via a REST gateway stub, and all account data is seeded in-process.

## Generate the system

```sh
cp -r ./single-agent.finance-analysis.whatsapp-fintech-agent  ~/my-projects/whatsapp-fintech-agent
cd ~/my-projects/whatsapp-fintech-agent
```

(Optional) Edit `SPEC.md` to adjust the HITL threshold (default $500), the seeded account fixtures, or the instruction set in `prompts/fintech-query-agent.md`.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FintechQueryAgent** — an AutonomousAgent that accepts a sanitized customer message and account context, calls balance/transaction tools, and returns a typed `AgentResponse`.
- **MessageWorkflow** — orchestrates sanitize-wait → respond → (optional) HITL-confirm → execute per inbound message.
- **MessageEntity** — an EventSourcedEntity holding the per-message lifecycle.
- **MessageSanitizer** — a Consumer that subscribes to `MessageReceived` events, redacts PII from the message text, and emits `MessageSanitized` back to the entity.
- **HitlGate** — an application-level HITL component that pauses the workflow when a transaction exceeds the value threshold and waits for an explicit human-approval command.
- **MessageView + MessageEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded account fixtures (the JSONL file under `src/main/resources/sample-events/accounts.jsonl` after generation).
- `SPEC.md §5` — extend `AgentResponse` with channel-specific fields (e.g., WhatsApp template ID, rich-card payload).
- `prompts/fintech-query-agent.md` — narrow the agent's tool set (a savings-only deployer would remove the transfer tools; an investment deployer would add portfolio-query tools).
- `eval-matrix.yaml` — raise or lower the HITL threshold by changing the `large_value_threshold_usd` key in `application.conf`.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer message containing an account balance inquiry is sanitized, answered by the agent, and the response appears in the UI.
2. A payment instruction that exceeds the threshold is intercepted by the HITL gate; a simulated operator approves it; the transaction completes and the customer response is sent.
3. A payment instruction that exceeds the threshold but is rejected by the operator lands in `HITL_REJECTED` and the customer receives a denial message.
4. A before-tool-call guardrail blocks a payment tool call whose destination account fails validation; the agent retries with a corrected tool call.
5. PII strings in the inbound message never appear in the LLM call log; only the redacted form does.

## License

Apache 2.0.
