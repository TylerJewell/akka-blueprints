# Akka Sample: Basic Tool-Calling Chatbot

A single ReAct-style agent receives a user message, decides which tools to call, calls them in a loop, and produces a final natural-language reply. Every response passes through a `before-agent-response` guardrail that checks for policy violations before the answer reaches the user.

Demonstrates the **single-agent** coordination pattern wired to one governance mechanism: a `before-agent-response` guardrail that intercepts every candidate reply, rejects those containing prohibited content, and forces the agent loop to try again within its iteration budget.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the tool implementations are in-process stubs and the agent's loop is self-contained.

## Generate the system

```sh
cp -r ./single-agent.cx-support.react-chatbot  ~/my-projects/react-chatbot
cd ~/my-projects/react-chatbot
```

(Optional) Edit `SPEC.md` to swap the seeded tool set (e.g., add a real product-catalog lookup, a CRM read, or a knowledge-base search) or to tighten the guardrail policy to your domain's content rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChatAgent** — an AutonomousAgent that accepts a chat turn, calls in-process tools in a loop until it has enough information, and returns a `ChatReply`.
- **ConversationEntity** — an EventSourcedEntity holding the per-conversation history: every user message and every agent reply.
- **ConversationWorkflow** — orchestrates the per-turn lifecycle: receive message → run agent → apply guardrail → record reply.
- **ReplyGuardrail** — a `before-agent-response` hook that validates every candidate reply against the configured policy before it leaves the agent loop.
- **ConversationView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded tool set (the in-process stubs in `src/main/resources/tools/` after generation) to point at real back-end APIs.
- `SPEC.md §5` — extend `ChatReply` with domain fields such as `citations`, `confidence`, or `suggestedActions`.
- `prompts/chat-agent.md` — narrow the agent persona and tool-use instructions (a retail deployer would constrain it to product FAQs; a support deployer to ticket resolution flows).
- `eval-matrix.yaml` — wire a real policy service by naming it under the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends a message requiring a tool call → the agent calls the tool, gets a result, and produces a final reply visible in the UI within 30 s.
2. The agent's first candidate reply is policy-violating (mock LLM path) → the `before-agent-response` guardrail rejects it → the agent retries → a clean reply reaches the user.
3. A conversation with three messages retains all three turns in the history visible on the right-pane detail.
4. The live list updates via SSE within 1 s of each state transition; the user never needs to refresh.

## License

Apache 2.0.
