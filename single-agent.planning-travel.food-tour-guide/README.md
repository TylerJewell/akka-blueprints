# Akka Sample: Gemma Food Tour Guide

A single culinary-tour agent reads a traveler's preferences and a city context, then generates a personalized, day-by-day food tour itinerary — including restaurants, street-food stops, market visits, and cultural context. The preferences ride into the agent as a task attachment; the city context is loaded from a seeded corpus.

Demonstrates the **single-agent** coordination pattern in the planning-travel domain. One `FoodTourAgent` (AutonomousAgent) builds the full itinerary; the surrounding components prepare its input, validate its structure, and score its coverage quality.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the city corpus lives in-process and Maps-style lookups are simulated.

## Generate the system

```sh
cp -r ./single-agent.planning-travel.food-tour-guide  ~/my-projects/food-tour-guide
cd ~/my-projects/food-tour-guide
```

(Optional) Edit `SPEC.md` to change the seeded city corpus (e.g., swap the three included cities for cities relevant to your region), or narrow the preference categories the agent reasons about.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **FoodTourAgent** — an AutonomousAgent that accepts traveler preferences and a city profile (passed as a task attachment) and returns a typed `TourItinerary`.
- **TourWorkflow** — orchestrates validate-input → generate → score per submitted tour request.
- **TourRequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **PreferenceValidator** — a Consumer that subscribes to `TourRequested` events, validates and normalizes traveler preferences, and emits `PreferencesValidated` back to the entity.
- **TourView + TourEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded city profiles (the JSONL file under `src/main/resources/sample-events/city-profiles.jsonl` after generation) for cities in your target market.
- `SPEC.md §5` — extend `TourItinerary` with domain-specific fields (e.g., `accessibility`, `dietaryAccommodations`, `estimatedBudgetPerDay`).
- `prompts/food-tour-agent.md` — narrow the agent's role (a luxury-travel deployer would constrain it to Michelin-starred venues; a backpacker platform to street-food and budget options).
- `eval-matrix.yaml` — wire a real Maps API by naming it under the location-verification mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits preferences for a city → preferences are validated → the agent generates a full itinerary → the itinerary appears in the UI.
2. The agent returns a malformed itinerary on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed itinerary lands.
3. Every recorded itinerary has a coverage score visible on the same UI card.
4. A request containing personal dietary restrictions or health details is normalized to category labels before the LLM sees it.

## License

Apache 2.0.
