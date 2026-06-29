# Akka Sample: Entity Workflow Chat

A multi-turn chat system where each conversation is a long-lived `Workflow` bound to an `EventSourcedEntity`. Conversation history accumulates in the entity; when it grows past a configurable token budget the workflow applies a continue-as-new compaction step, summarising older turns and discarding them from the live context window while keeping the full turn log on the entity for audit.

Demonstrates the **single-agent** coordination pattern in the customer-support domain. One `ConversationAgent` (AutonomousAgent) handles every user turn. Two governance mechanisms sit around it: a PII sanitizer that strips customer identifiers from incoming messages before the agent ever sees them, and a user-driven human-in-the-loop gate that makes every turn an explicit control input rather than a fire-and-forget call.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — conversation state lives inside the Akka runtime.

## Generate the system

```sh
cp -r ./single-agent.cx-support.akka-entity-chat-workflow  ~/my-projects/entity-chat
cd ~/my-projects/entity-chat
```

(Optional) Edit `SPEC.md` to change the compaction threshold (default 4 000 tokens) or the seeded support scenarios.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ConversationAgent** — an AutonomousAgent that receives a sanitized user message and the compacted conversation context, and returns a typed `AgentReply`.
- **ChatWorkflow** — one long-lived workflow per conversation. Steps: `sanitizeStep` → `agentStep` → `recordStep` → optional `compactStep`.
- **ConversationEntity** — an EventSourcedEntity holding the per-conversation turn log, compaction summaries, and current status.
- **MessageSanitizer** — a Consumer that subscribes to `MessageReceived` events, strips PII from the raw message, and emits `MessageSanitized` back to the entity.
- **ConversationView + ChatEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the three seeded support scenarios (billing dispute, password reset, order status) with scenarios relevant to your product.
- `SPEC.md §5` — extend `AgentReply` with domain-specific fields (e.g., `caseId`, `escalationReason`, `resolvedCategory`).
- `prompts/conversation-agent.md` — tighten the agent's persona and allowed response scope to match your support vertical.
- `eval-matrix.yaml` — swap the regex-based PII scrubber for a purpose-built redaction library relevant to your data types.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user opens a conversation and sends three turns → each reply appears within the compaction threshold; the full turn log persists on the entity.
2. A conversation that exceeds the token budget triggers compaction → older turns are summarised → the agent's next reply refers to the summary correctly.
3. PII strings in user messages never appear in the agent's context; only the redacted form does.
4. Every turn the user sends is a first-class workflow resumption — there is no background polling loop driving the agent.

## License

Apache 2.0.
