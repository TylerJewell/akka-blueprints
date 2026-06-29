# Akka Sample: Foursquare Location Agent

A single location-discovery agent accepts a natural-language search query and a GPS coordinate, queries place data, and returns a ranked list of nearby places with PII-free context. User coordinates are sanitized before the agent receives them; the agent's recommendation is scored for relevance quality on every response.

Demonstrates the **single-agent** coordination pattern in the planning-travel domain. One `LocationDiscoveryAgent` (AutonomousAgent) carries the entire recommendation decision; surrounding components prepare its input, guard its output structure, and audit the quality of every result.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Place data is seeded in-process; the agent's tool calls are simulated against the bundled dataset.

## Generate the system

```sh
cp -r ./single-agent.planning-travel.location-discovery-agent  ~/my-projects/location-discovery-agent
cd ~/my-projects/location-discovery-agent
```

(Optional) Edit `SPEC.md` to swap the seeded place dataset for your own (e.g., substitute venue categories, geographic areas, or the scoring weights in `EvaluationScorer`).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **LocationDiscoveryAgent** — an AutonomousAgent that receives a search query plus coordinate context as a task attachment and returns a typed `PlaceRecommendation`.
- **DiscoveryWorkflow** — orchestrates sanitize-wait → discover → eval per submitted search.
- **SearchEntity** — an EventSourcedEntity holding the per-search lifecycle.
- **CoordinateSanitizer** — a Consumer that subscribes to `SearchSubmitted` events, redacts exact GPS coordinates to a bounding-box token, and emits `CoordinateSanitized` back to the entity.
- **SearchView + SearchEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded place categories and venue dataset for your own.
- `SPEC.md §5` — extend `PlaceRecommendation` with domain-specific fields (e.g., `accessibilityRating`, `priceLevel`, `openNow`).
- `prompts/location-discovery.md` — narrow the agent's role (a travel deployer would constrain it to hotels and transit; a dining deployer to restaurant discovery with dietary filters).
- `eval-matrix.yaml` — wire a real coordinate-privacy vault by naming it under the sanitizer mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a search query with coordinates → coordinates are sanitized → the agent discovers places → ranked recommendations appear in the UI.
2. The agent returns a malformed recommendation on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed recommendation lands.
3. Every recorded recommendation has an on-decision eval score visible on the same UI card.
4. Exact GPS coordinates submitted by the user never appear in the LLM call log; only the bounding-box token does.

## License

Apache 2.0.
