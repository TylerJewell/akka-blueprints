# Akka Sample: Video Games Sales Assistant

A single-agent sales assistant accepts a customer query about video games — genre, platform, budget, or a specific title — and returns a structured `SalesRecommendation`: a ranked list of game suggestions with a rationale per suggestion, a confidence score, and an upsell opportunity. A `before-agent-response` guardrail runs on every agent turn to ensure the response is well-formed before it reaches the customer-facing UI.

Demonstrates the **single-agent** coordination pattern wired with one governance mechanism: a `before-agent-response` guardrail that validates the recommendation structure, ensuring every suggested title references a known catalog entry and every confidence score is in range, before the recommendation is written to state and surfaced on the UI.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the game catalog lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.sales-marketing.games-sales  ~/my-projects/games-sales
cd ~/my-projects/games-sales
```

(Optional) Edit `SPEC.md` to adjust the seeded game catalog, add a specific platform or genre filter, or narrow the agent's recommendation rules.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **SalesAssistantAgent** — an AutonomousAgent that accepts a customer query (preferences, platform, budget) as task instructions plus an optional cart-context attachment, and returns a typed `SalesRecommendation`.
- **SessionWorkflow** — orchestrates the per-session lifecycle: query received → agent call → guardrail → recommendation recorded → optional follow-up.
- **SessionEntity** — an EventSourcedEntity holding the per-session lifecycle and recommendation history.
- **RecommendationView + SalesEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded catalog for your own (the JSONL file under `src/main/resources/sample-events/game-catalog.jsonl` after generation).
- `SPEC.md §5` — extend `SalesRecommendation` with retailer-specific fields (e.g., `promotionCode`, `bundleEligible`, `loyaltyPoints`).
- `prompts/sales-assistant.md` — narrow the agent's role (a family-oriented retailer would constrain it to age-appropriate titles; a specialty store to a specific genre).
- `eval-matrix.yaml` — wire a real catalog lookup by naming it under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer submits a query → the agent produces a structured recommendation → the recommendation appears in the UI with confidence scores and rationale per title.
2. The agent returns a malformed recommendation on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed recommendation lands.
3. A recommendation referencing a catalog entry not in the seeded list is rejected by the guardrail; the customer never sees the invalid title.
4. A follow-up query in the same session inherits the prior recommendation context and produces a refined result.

## License

Apache 2.0.
