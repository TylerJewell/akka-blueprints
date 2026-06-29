# Akka Sample: Maps Trip Planner

A single trip-planner agent takes a natural-language travel request, calls the Google Maps MCP to resolve real places and distances, and returns a structured day-by-day itinerary. The request rides into the agent as a task; the Maps MCP tools are the only external calls the agent is permitted to make.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a destination-safety screener that runs before the agent ever sees the request, a `before-agent-response` guardrail that validates the itinerary's structure and geocoding references, and an on-decision evaluator that scores every itinerary for coverage and realism.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The Maps MCP integration is simulated in-process. A deployer who wants live geocoding points the agent at a real Maps MCP host by updating `application.conf`; the blueprint runs without it.

## Generate the system

```sh
cp -r ./single-agent.planning-travel.maps-trip-planner  ~/my-projects/maps-trip-planner
cd ~/my-projects/maps-trip-planner
```

(Optional) Edit `SPEC.md` to change the seeded trip scenarios (e.g., swap the European city-break examples for domestic road-trip examples) or to extend `ItineraryDay` with lodging fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TripPlannerAgent** — an AutonomousAgent that accepts a travel request (origin, destination, dates, preferences) and calls Maps MCP tools to produce a typed `TripItinerary`.
- **PlanningWorkflow** — orchestrates screen-wait → plan → eval per submitted request.
- **TripRequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **RequestScreener** — a Consumer that subscribes to `TripRequestSubmitted` events, checks destination safety flags, and emits `RequestScreened` back to the entity.
- **ItineraryView + PlanningEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded trip scenarios for your own (the JSONL file under `src/main/resources/sample-events/trip-scenarios.jsonl` after generation).
- `SPEC.md §5` — extend `ItineraryDay` with lodging fields, transport booking links, or estimated costs.
- `prompts/trip-planner.md` — narrow the agent's role (a business-travel deployer would constrain it to approved vendors; a family-travel deployer would add child-friendly filtering).
- `eval-matrix.yaml` — wire a real Maps MCP endpoint by naming the host and API-key reference in the implementation paragraph for the geocoding mechanism.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a trip request → it is screened → planned → the itinerary appears in the UI with geocoded place references.
2. The agent returns a malformed itinerary on the first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed itinerary lands.
3. Every recorded itinerary has an on-decision eval score visible on the same UI card.
4. A request for a destination on the screener's restricted list is blocked before the agent call; the entity records a `RequestBlocked` event and the UI shows the card in BLOCKED state.

## License

Apache 2.0.
