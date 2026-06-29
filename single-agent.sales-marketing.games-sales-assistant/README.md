# Akka Sample: Video Games Sales Assistant

A retail sales-assistant agent answers shopper questions about a video games catalog, recommends titles, checks order history, and surfaces promotions. Every customer-facing recommendation passes through a `before-agent-response` guardrail that strips unsafe content and rejects off-catalog references before they reach the shopper.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that validates every agent response before it is returned to the caller.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the catalog and order data live in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.sales-marketing.games-sales-assistant  ~/my-projects/games-sales-assistant
cd ~/my-projects/games-sales-assistant
```

(Optional) Edit `SPEC.md` to expand the seeded catalog (swap genres, add platform variants, adjust pricing tiers) or to tighten the guardrail's allowed-content policy.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SalesAssistantAgent** — an AutonomousAgent that accepts a shopper's natural-language question plus their session context and returns a typed `AssistantResponse`.
- **SessionWorkflow** — tracks each conversation session; one workflow per `sessionId` with steps: `initStep` → `assistStep` (loops per turn) → `closeStep`.
- **SessionEntity** — an EventSourcedEntity holding per-session state: catalog lookups performed, recommendations made, and order queries issued.
- **CatalogConsumer** — a Consumer that subscribes to `CatalogUpdated` events emitted by a seeded catalog loader and hydrates the in-process catalog index.
- **SessionView + SessionEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded game catalog for your own (the JSONL file under `src/main/resources/sample-events/catalog.jsonl` after generation).
- `SPEC.md §5` — extend `GameTitle` with platform-specific fields (e.g., `ageRating`, `multiplayerSupport`, `dlcCount`).
- `prompts/sales-assistant.md` — restrict the agent's persona (a storefront deployer might constrain it to their own first-party titles; a marketplace deployer might enable price-comparison logic).
- `eval-matrix.yaml` — add an on-decision evaluator if the deployer wants every recommendation scored for catalog accuracy.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A shopper asks for an action RPG under $40 → the agent returns at most 5 catalog matches with price, platform, and a one-sentence pitch.
2. The agent's first response on a turn references a title that is not in the catalog — the `before-agent-response` guardrail rejects it; the agent retries and returns only in-catalog titles.
3. A shopper asks about their past orders → the agent returns the correct order history from `SessionEntity`.
4. A guardrail trigger that fires on inappropriate content blocks the response and the session card shows a `GUARDRAIL_REJECTED` status chip rather than a recommendation.

## License

Apache 2.0.
