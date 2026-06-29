# Akka Sample: Retail Customer Service

A single customer-service agent handles product questions, order lookups, and order modifications for a plant retailer. The agent uses tools to read the product catalog, retrieve orders, and apply order changes. Two guardrails bracket every tool invocation and every customer-facing reply: one validates that order-modifying tool calls are allowed given the order's current state, and one ensures the agent's response is appropriate for a public-facing channel before it is sent to the customer.

Demonstrates the **single-agent** coordination pattern in the cx-support domain with two guardrails wired around the same agent: a `before-tool-call` guardrail on order-modification tools and a `before-agent-response` guardrail on all outbound replies.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Product catalog and order data live in-process; the agent's tool calls run against seeded in-memory fixtures.

## Generate the system

```sh
cp -r ./single-agent.cx-support.retail-customer-service  ~/my-projects/retail-customer-service
cd ~/my-projects/retail-customer-service
```

(Optional) Edit `SPEC.md` to point at a different product catalog (e.g., switch from the Cymbal Home & Garden plant catalog to a general garden supply catalog) or extend the order-status rules governing when modifications are allowed.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CustomerServiceAgent** — an AutonomousAgent that handles a customer session. It receives the customer's message plus their order history (as a task attachment) and returns a typed `AgentReply` with a response message and any order change requested.
- **SessionWorkflow** — orchestrates session creation → agent turn → tool resolution → reply delivery per customer session.
- **SessionEntity** — an EventSourcedEntity holding per-session conversation history and state.
- **OrderEntity** — an EventSourcedEntity representing a single order: status, line items, and modification history.
- **OrderView + SessionView + CustomerServiceEndpoint + AppEndpoint** — read models, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the Cymbal Home & Garden seeded catalog for your own product JSONL under `src/main/resources/sample-events/products.jsonl` after generation.
- `SPEC.md §5` — extend `OrderModification` with retailer-specific fields (e.g., `giftMessage`, `substituteOnOutOfStock`).
- `prompts/customer-service-agent.md` — narrow the agent's persona, restrict product categories it can discuss, or add brand voice rules.
- `eval-matrix.yaml` — swap the in-process order-validation logic for a call to an external OMS by naming it under the before-tool-call guardrail's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer asks about a plant's care requirements → the agent looks up the product and replies with accurate, catalog-grounded detail.
2. A customer requests an address change on a pending order → the before-tool-call guardrail allows it (order is `PENDING`) → the modification lands and the confirmation appears in the UI.
3. A customer requests a cancellation on an order that has already shipped → the before-tool-call guardrail blocks the tool call → the agent explains why in its reply without exposing internal error detail.
4. The agent produces a response containing a competitor brand name → the before-agent-response guardrail rejects it → the agent rewrites the reply without the forbidden content.

## License

Apache 2.0.
