# Akka Sample: Multi-Model Chatbot

A single-agent customer support chatbot that routes user messages to one of several LLM providers and streams replies back over SSE. The active provider is selectable at runtime; all chat turns are stored on an event-sourced entity for audit. Response content is moderated by a `before-agent-response` guardrail before any text leaves the agent loop, and PII in user messages is redacted before the model ever sees the content.

Demonstrates the **single-agent** coordination pattern in the cx-support domain, wired with two governance mechanisms: a PII sanitizer that scrubs user input before the agent call, and a `before-agent-response` guardrail that blocks harmful or policy-violating replies.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. All chat history and session state live in-process.

## Generate the system

```sh
cp -r ./single-agent.cx-support.multi-model-chatbot  ~/my-projects/multi-model-chatbot
cd ~/my-projects/multi-model-chatbot
```

(Optional) Edit `SPEC.md` to point the chatbot at your own support knowledge base or to restrict the allowed provider list to a single model.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChatAgent** — an AutonomousAgent that receives a sanitized user message plus conversation history and returns a typed `ChatReply`.
- **ChatWorkflow** — orchestrates sanitize-wait → reply → moderate per submitted message.
- **ConversationEntity** — an EventSourcedEntity holding the full per-session message log.
- **MessageSanitizer** — a Consumer that subscribes to `MessageReceived` events, redacts PII from the user text, and emits `MessageSanitized` back to the entity.
- **ReplyGuardrail** — a `before-agent-response` hook that blocks replies containing harmful content, off-topic material, or personally identifying information before they leave the agent loop.
- **ConversationView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded support scenarios for your own product domain.
- `SPEC.md §5` — extend `ChatReply` with domain-specific fields (e.g., `ticketId`, `escalationFlag`, `confidenceScore`).
- `prompts/chat-agent.md` — narrow the agent's persona (a SaaS deployer would restrict it to their product's feature set; a healthcare deployer would add HIPAA-appropriate refusal wording).
- `eval-matrix.yaml` — wire a real moderation service under the guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user sends a support message → it is sanitized → the agent replies → the reply appears in the chat UI.
2. The agent produces a policy-violating reply on first try → the `before-agent-response` guardrail rejects it → the agent retries → a clean reply lands.
3. PII strings submitted in chat messages never appear in the LLM call log; only the redacted form does.
4. Switching the active provider mid-session produces a reply from the new provider within the same conversation.

## License

Apache 2.0.
