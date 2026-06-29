# Akka Sample: Realtime Conversational Agent

A single conversational agent handles real-time voice and text interactions with customers. Each session receives a greeting, a turn-by-turn conversation loop, and a session summary when the customer disconnects. Every agent response passes through a `before-agent-response` guardrail that blocks unsafe, off-topic, or policy-violating content before it reaches the customer.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a response-safety guardrail that runs before the agent's answer leaves the loop, ensuring customer-facing output meets the operator's content policy on every turn.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — there is no telephony backend, no media server, no external CRM. Conversation sessions live in-process; voice is simulated as text I/O in the embedded UI.

## Generate the system

```sh
cp -r ./single-agent.cx-support.realtime-voice  ~/my-projects/realtime-voice
cd ~/my-projects/realtime-voice
```

(Optional) Edit `SPEC.md` to wire a different greeting script, narrow the agent's persona (e.g., an airline check-in bot, a bank account inquiry bot), or replace the seeded FAQ knowledge base with your own.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationalAgent** — an AutonomousAgent that maintains a multi-turn conversation with a customer and returns a typed `AgentTurn` (the agent's reply plus safety metadata) on each turn.
- **SessionWorkflow** — orchestrates greet → converse loop → summarize per session.
- **SessionEntity** — an EventSourcedEntity holding the per-session lifecycle and the full turn history.
- **ResponseGuardrail** — the `before-agent-response` hook that checks each candidate reply against the operator's content policy before it leaves the agent loop.
- **TurnSummarizer** — a deterministic post-session scorer (no LLM call) that produces a session summary and a quality score.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded FAQ knowledge base for your own (`src/main/resources/sample-events/faq-entries.jsonl` after generation).
- `SPEC.md §5` — extend `AgentTurn` with domain-specific fields (e.g., `intentLabel`, `escalationRequested`, `sentimentScore`).
- `prompts/conversational-agent.md` — narrow the agent's persona (a healthcare deployer would constrain it to appointment scheduling; a financial deployer to account balance inquiries).
- `eval-matrix.yaml` — tighten the `ResponseGuardrail` policy by adding categories (e.g., competitor-mention blocking, regulated-advice detection).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer opens a session → the agent greets them → the customer sends three messages → the session summary appears in the UI.
2. The agent produces a policy-violating candidate reply on one turn → the `before-agent-response` guardrail rejects it → the agent retries → a safe reply reaches the customer. The unsafe text never lands in `SessionEntity`.
3. Every session ends with a session-quality score visible on the session card.
4. A session that is still in progress shows live turn-by-turn updates in the UI via SSE; a browser joining mid-session sees all prior turns from the view.

## License

Apache 2.0.
