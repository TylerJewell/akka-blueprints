# Akka Sample: Restaurant Assistant

A single customer-service agent handles menu queries, table reservations, and order placement for a restaurant. The agent answers questions from a menu catalog, accepts reservation requests, and submits orders — with two guardrails ensuring safe, accurate customer-facing responses before they are delivered and before any write operation is committed.

Demonstrates the **single-agent** coordination pattern in the cx-support domain. One `RestaurantAssistantAgent` (AutonomousAgent) drives all customer interactions; the surrounding components manage session state and audit the agent's decisions.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the menu catalog and seeded reservation/order data live in-process.

## Generate the system

```sh
cp -r ./single-agent.cx-support.restaurant-assistant  ~/my-projects/restaurant-assistant
cd ~/my-projects/restaurant-assistant
```

(Optional) Edit `SPEC.md` to point at your own menu items (update the JSONL seed under `src/main/resources/sample-events/menu-items.jsonl`) or adjust reservation-window constraints in Section 3.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RestaurantAssistantAgent** — an AutonomousAgent that accepts customer messages and the current session context (open reservation, active order, selected menu items) and returns a typed `AssistantResponse`.
- **SessionWorkflow** — orchestrates the customer session: opens on first message, routes to the agent, and commits reservation or order writes after guardrail approval.
- **SessionEntity** — an EventSourcedEntity holding per-session lifecycle and accumulated cart/reservation state.
- **MenuConsumer** — a Consumer that projects `MenuItemUpdated` events into the in-process menu catalog available to the agent.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — adjust the seeded menu (items, prices, dietary flags) in the JSONL seed file reference.
- `SPEC.md §5` — add fields to `ReservationRequest` (party size limits, deposit required) or `OrderLineItem` (modifiers, allergens).
- `prompts/restaurant-assistant.md` — narrow the agent's persona or restrict which actions it may suggest (a fast-casual deployer would remove reservation logic; a fine-dining deployer might add wine-pairing suggestions).
- `eval-matrix.yaml` — strengthen the `before-tool-call` guardrail to enforce business rules (maximum party size, minimum order total, blackout dates).

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer asks about a menu item → the agent returns accurate details from the in-process catalog within one turn.
2. A customer requests a reservation → the `before-tool-call` guardrail validates the request (date in the future, party size within range) before the write lands.
3. A customer places an order → each line item is validated against the live catalog before the order is committed.
4. The agent produces a response that fails the `before-agent-response` guardrail (e.g., a hallucinated menu item) → the agent retries within its iteration budget; the invalid response never reaches the customer.

## License

Apache 2.0.
