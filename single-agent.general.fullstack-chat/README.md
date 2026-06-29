# Akka Sample: Gemini Fullstack Chat

A single-agent fullstack chat application where a user sends a message, one AI agent generates a reply, and both the message history and agent responses stream back to the browser in real time. The agent runs inside a durable Akka backend; the embedded UI is a self-contained HTML page served from the same process.

Demonstrates the **single-agent** coordination pattern in the general domain. One `ChatAgent` (AutonomousAgent) handles every reply; a `ConversationEntity` holds the durable message history; a `ConversationWorkflow` manages the per-turn lifecycle; and a streaming View feeds the UI over Server-Sent Events.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the message store is in-process and the browser connects directly to the service's HTTP port.

## Generate the system

```sh
cp -r ./single-agent.general.fullstack-chat  ~/my-projects/fullstack-chat
cd ~/my-projects/fullstack-chat
```

(Optional) Edit `SPEC.md` to change the agent persona (e.g., narrow the system prompt in `prompts/chat-agent.md` to a domain-specific assistant) or extend `ChatMessage` with metadata fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ChatAgent** — an AutonomousAgent that receives a conversation turn (the user's message plus recent history as a task attachment) and returns a typed `AgentReply`.
- **ConversationWorkflow** — per-turn lifecycle: validate → generate → record. Bounded by step timeouts so long LLM calls never stall the system.
- **ConversationEntity** — an EventSourcedEntity holding the per-conversation message history (user messages and agent replies).
- **ConversationView** — a read model streaming all turns to the UI via SSE.
- **ChatEndpoint + AppEndpoint** — REST/SSE API and the embedded single-file UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded conversation starters for your own (the JSONL file under `src/main/resources/sample-events/seed-conversations.jsonl` after generation).
- `SPEC.md §5` — extend `ChatMessage` with metadata (e.g., `language`, `sentiment`, `topicTag`).
- `prompts/chat-agent.md` — rewrite the system prompt to specialise the agent (customer-support, coding assistant, knowledge-base Q&A, etc.).
- `eval-matrix.yaml` — this baseline has no governance controls; a deployer adding content-safety or PII concerns should add controls here before generating.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user opens the App UI, types a message, and within 30 s receives an agent reply that appears in the conversation.
2. A new conversation started independently runs in isolation — messages from one conversation do not appear in another.
3. The SSE stream delivers real-time status updates so the UI transitions smoothly from AWAITING_REPLY to REPLY_RECORDED without polling.
4. The conversation history is durable: refreshing the page and re-fetching the conversation returns all prior turns.

## License

Apache 2.0.
