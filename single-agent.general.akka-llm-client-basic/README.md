# Akka Sample: Conversation API LLM Client

A minimal example that calls an LLM through Akka's Conversation API and returns the model's reply to the caller. No tools, no multi-step agent loop — just a clean provider-portable request-reply call with a single response guardrail applied before the output leaves the system.

Demonstrates the **single-agent** coordination pattern at its most direct: one `ConversationAgent` receives a prompt, forwards it to the configured model provider, and the response passes through a `before-agent-response` guardrail before the endpoint returns it. Subsequent quickstart blueprints that introduce durable state and tool calls build on the same Conversation API interface established here.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box.

## Generate the system

```sh
cp -r ./single-agent.general.akka-llm-client-basic  ~/my-projects/llm-client-basic
cd ~/my-projects/llm-client-basic
```

(Optional) Edit `SPEC.md` to change the system prompt loaded from `prompts/conversation-agent.md`, or to adjust the `before-agent-response` guardrail checks in Section 8.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationAgent** — an AutonomousAgent that accepts a user prompt and a conversation history, calls the configured model provider, and returns a typed `ConversationReply`.
- **ConversationEntity** — an EventSourcedEntity holding per-session turn history and status.
- **ConversationView + ConversationEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- **ReplyGuardrail** — a `before-agent-response` check that runs on every candidate reply before it reaches the caller.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded example prompts (the JSONL file under `src/main/resources/sample-events/seed-prompts.jsonl` after generation).
- `SPEC.md §5` — extend `ConversationReply` with fields specific to your use case (e.g., `confidenceLevel`, `citations`).
- `prompts/conversation-agent.md` — narrow the agent's role and tone for your deployment context.
- `eval-matrix.yaml` — wire the guardrail to additional content-policy checks relevant to your domain.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends a prompt → the system calls the model → the reply appears in the UI within the timeout.
2. The agent's first response fails the guardrail → the guardrail rejects it → the agent retries → a passing reply returns to the caller.
3. An empty prompt is submitted → the system returns a structured validation error before any model call is made.
4. Turn history is preserved across multiple prompts in the same session.

## License

Apache 2.0.
