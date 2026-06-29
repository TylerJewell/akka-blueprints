# Akka Sample: Voyage Virtuoso Multi-Agent

A travel-planning supervisor delegates work to four specialty subagents — one per travel pillar — running **in parallel**, then merges their outputs into one premium itinerary recommendation. Demonstrates the **delegation-supervisor-workers** coordination pattern with an output guardrail protecting customer-facing content.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound request stream and the travel-pillar tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.planning-travel.voyage-virtuoso-team  ~/my-projects/voyage-virtuoso
cd ~/my-projects/voyage-virtuoso
```

(Optional) Edit `SPEC.md` to change the simulated travel requests, model provider, or itinerary schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ItineraryDirector** — AutonomousAgent that decomposes a travel request into four parallel work items and assembles the final itinerary from the specialists' outputs.
- **FlightSpecialist** — AutonomousAgent that identifies routing options and fare classes for the requested trip.
- **AccommodationSpecialist** — AutonomousAgent that selects lodging properties matching the traveller's tier and preferences.
- **ExperienceSpecialist** — AutonomousAgent that curates activities, dining, and cultural highlights at the destination.
- **LogisticsSpecialist** — AutonomousAgent that plans ground transport, transfers, and visa / entry requirements.
- **ItineraryWorkflow** — Workflow that fans the work out to the four specialists in parallel, then asks ItineraryDirector to assemble the itinerary.
- **TravelRequestEntity** — EventSourcedEntity holding the full request lifecycle.
- **TravelRequestView** — projection the UI streams via SSE.
- **TravelEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the simulated travel scenarios the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ItineraryPlan` record fields (e.g., add `carbonFootprintKg`).
- `prompts/flight-specialist.md` — narrow the agent to a specific alliance or cabin class.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real GDS or booking API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a travel request → request enters `PLANNING`, then `IN_PROGRESS`, then `ASSEMBLED`.
2. Specialists fail-fast → if any specialist times out, the request enters `PARTIAL` with whichever pillar outputs arrived.
3. The output guardrail catches a fabricated fare or property and moves the request to `BLOCKED`.
4. Wait after a successful assembly; the request row shows an eval quality score.

## License

Apache 2.0.
