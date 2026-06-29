# Akka Sample: Customer Service Agent with Tools

A single AI agent handles customer support queries by calling tools — look up orders, fetch account details, check product inventory, submit refund requests — and returns a grounded, structured reply. The agent operates in a conversational loop; each customer message drives one task execution with tool access.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that strips customer identifiers from the agent's log output, a `before-tool-call` guardrail that gates write operations and data-access tool calls, and a `before-agent-response` guardrail that screens every customer-facing reply before it reaches the conversation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — customer records, orders, and product catalog live in-process with seeded data; the agent's tool calls operate against that in-process store.

## Generate the system

```sh
cp -r ./single-agent.cx-support.customer-service-tool-agent  ~/my-projects/cx-agent
cd ~/my-projects/cx-agent
```

(Optional) Edit `SPEC.md` to extend the tool set (e.g., add a `scheduleTechnicianVisit` tool for field-service support, or connect the order-lookup tool to a real OMS endpoint).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SupportAgent** — an AutonomousAgent that accepts a customer message, executes tool calls in a loop, and returns a typed `AgentReply`.
- **ConversationWorkflow** — orchestrates the per-session lifecycle: open → active → resolved / escalated.
- **ConversationEntity** — an EventSourcedEntity holding the per-session conversation history and status.
- **ReplyScreener** — a Consumer that subscribes to `AgentReplied` events, runs the before-agent-response screen, and emits `ReplyScreened` or `ReplyBlocked` back to the entity.
- **ConversationView + SupportEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded customer records and order catalog for your own (the JSONL files under `src/main/resources/sample-data/` after generation).
- `SPEC.md §5` — add domain-specific fields to `AgentReply` (e.g., `caseNumber`, `escalationReason`, `resolvedAt`).
- `prompts/support-agent.md` — narrow the agent's persona and tool authorization scope (a telco deployer would allow `suspendLine` but not `closeAccount`; a retailer might disable `processRefund` above a threshold).
- `eval-matrix.yaml` — wire a real PII detector (e.g., a presidio-style library) by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer sends a message asking for order status → the agent calls `lookupOrder`, returns a grounded reply, and the conversation appears in the UI.
2. The agent attempts to call `processRefund` on an order whose amount exceeds the auto-approve threshold → the `before-tool-call` guardrail blocks the call and the agent escalates instead.
3. The agent drafts a reply containing a raw credit-card number → the `before-agent-response` guardrail rejects it; the agent retries without the PAN.
4. Customer PII strings logged during a session never appear in plaintext in the structured log output; only redacted tokens do.

## License

Apache 2.0.
