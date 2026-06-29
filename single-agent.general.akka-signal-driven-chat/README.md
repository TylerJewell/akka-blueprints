# Akka Sample: with Signals & Queries

A single agent runs inside a long-lived Workflow. After the workflow starts, external callers can **signal** it to inject additional user prompts mid-flight and **query** it to read the agent's current state — enabling interactive, multi-turn sessions without reopening a new workflow each time.

Demonstrates the **single-agent** coordination pattern extended with Workflow signals and queries. An inbound-signal guardrail validates each incoming prompt before it drives the agent. A human-in-the-loop signal gate lets an operator pause execution and inject a corrective prompt at any point during the session.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box.

## Generate the system

```sh
cp -r ./single-agent.general.akka-signal-driven-chat  ~/my-projects/signal-driven-chat
cd ~/my-projects/signal-driven-chat
```

(Optional) Edit `SPEC.md` to change the agent's role — for example, point it at a domain-specific system prompt in `prompts/chat-agent.md`, or extend `SessionState` with additional context fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChatAgent** — an AutonomousAgent that processes a prompt and returns a typed `AgentReply`. Each task carries the full conversation so far as context.
- **ChatSessionWorkflow** — a long-running Workflow that accumulates turns. Accepts `addTurnSignal` to inject new prompts; exposes `getSessionQuery` to inspect current state without side effects.
- **ChatSessionEntity** — an EventSourcedEntity holding the per-session turn log and status.
- **SignalValidator** — validates inbound signal payloads before they drive the agent (before-agent-invocation guardrail).
- **SessionView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded system-prompt scenario (the three seeds are: open Q&A, code-review assistant, meeting-notes summariser).
- `SPEC.md §5` — extend `TurnRecord` with domain-specific metadata (e.g., `topic`, `citedDocuments`, `confidenceScore`).
- `prompts/chat-agent.md` — replace the generic assistant persona with a domain expert.
- `eval-matrix.yaml` — wire a real content-policy library under the inbound-signal guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user opens a session, sends a first prompt, and receives a reply — all visible in real time via SSE.
2. A user sends a follow-up signal mid-session; the workflow picks it up, the agent processes the accumulated context, and the updated session appears in the UI.
3. An operator queries the workflow for its current state without triggering a new agent turn; the response is instantaneous.
4. A signal with a blank prompt body is rejected by the guardrail and never reaches the agent.

## License

Apache 2.0.
