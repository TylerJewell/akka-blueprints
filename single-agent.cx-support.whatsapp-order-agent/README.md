# Akka Sample: WhatsApp Order Agent

A single conversational agent handles order management over WhatsApp. Customers send free-text messages — place a new order, check status, modify quantities, or cancel — and the agent reads the product catalog and order database, executes the right tool call, and replies in natural language. The agent never writes to external systems without passing a `before-tool-call` guardrail, phone numbers and delivery addresses are redacted before the conversation context is stored in any log, and large orders (above a configurable threshold) are held for human confirmation before the agent proceeds.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a `before-tool-call` guardrail that inspects every write before it is executed, a PII sanitizer that strips contact and payment details from the stored conversation context, and an application-level HITL gate that routes high-value orders to a human queue.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the product catalog and order database live in-process, seeded from JSONL files in `src/main/resources/sample-events/`.

## Generate the system

```sh
cp -r ./single-agent.cx-support.whatsapp-order-agent  ~/my-projects/whatsapp-order-agent
cd ~/my-projects/whatsapp-order-agent
```

(Optional) Edit `SPEC.md` to change the order threshold for the HITL gate, the product catalog, or the currency/locale of the UI.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **OrderAgent** — an AutonomousAgent that maintains a conversation with one customer per session, dispatches tool calls to look up products, create orders, modify orders, and check status, and returns a natural-language reply each turn.
- **OrderWorkflow** — coordinates the session lifecycle: idle → active → awaiting-hitl (for large orders) → completing.
- **OrderEntity** — an EventSourcedEntity holding the per-order lifecycle and history.
- **ConversationSanitizer** — a Consumer that subscribes to `TurnCompleted` events, redacts PII from the stored conversation context, and emits `ConversationSanitized` back to the entity.
- **OrderView + OrderEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the seeded product catalog (`src/main/resources/sample-events/products.jsonl`) to your domain.
- `SPEC.md §5` — extend `OrderRequest` with channel-specific fields (e.g., `whatsappMessageId`, `locale`).
- `prompts/order-agent.md` — narrow the agent's role, tone, or escalation policy.
- `eval-matrix.yaml` — raise or lower the HITL threshold under the `hitl` control's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer sends "order 2 units of SKU-1001" → the agent confirms availability, calls the create-order tool, and replies with an order confirmation number.
2. The agent attempts to call a write tool without passing the guardrail → the guardrail blocks it → the agent re-plans and tries an allowed variation → the order lands.
3. A customer places an order above the HITL threshold → the conversation pauses, a human-review card appears in the UI → an operator approves it → the agent resumes and confirms.
4. The stored conversation context for a turn containing a phone number and delivery address shows `[REDACTED-PHONE]` and `[REDACTED-ADDRESS]` in the sanitized log; the raw turn is retained on the entity for audit only.

## License

Apache 2.0.
