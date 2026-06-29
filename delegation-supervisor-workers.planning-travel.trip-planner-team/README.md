# Akka Sample: Trip Planner Team

A supervisor delegates a trip request to three worker agents — a local-expert that gathers city insights from a simulated search-and-scrape surface, an itinerary planner, and a logistics planner — then assembles a grounded travel plan.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One AI key option (any one is enough; if none is set the generator offers a mock provider):
  - `ANTHROPIC_API_KEY`, or
  - `OPENAI_API_KEY`, or
  - `GOOGLE_AI_GEMINI_API_KEY`.
- Host software for this integration form (**Runs out of the box**): None. The search and scrape surfaces are modeled inside the same service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.planning-travel.trip-planner-team  ~/my-projects/trip-planner-team
cd ~/my-projects/trip-planner-team
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or destination defaults.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. Section 12 of `SPEC.md` instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` and to print the listening URL when the service is up.

## What you'll get

- A `TripPlanningWorkflow` (supervisor) that delegates to three worker agents in sequence.
- `LocalExpertAgent`, `ItineraryAgent`, `LogisticsAgent` — request/response agents.
- A `TripEntity` event-sourced lifecycle and a `TripsView` read model streamed over SSE.
- An in-process search + scrape surface the local-expert agent calls, with a URL/robots guard before the scrape tool runs.
- A grounding check on the itinerary before it reaches the user.
- A background request simulator and a single self-contained web UI with five tabs.

## Customise before generating

- `SPEC.md` Section 1 — system name and one-line pitch.
- `SPEC.md` Section 11 — model provider and port (`9385`).
- `prompts/*.md` — agent behavior and tone.
- `risk-survey.yaml` — deployer-specific fields marked `TO_BE_COMPLETED_BY_DEPLOYER`.

## What gets validated

The user journeys in `reference/user-journeys.md`:

- Submit a destination and watch insights, itinerary, and logistics fill in.
- A scrape against a non-allowlisted URL is blocked before the tool runs.
- An itinerary that references a place absent from the gathered insights is blocked before it reaches the user.
- The background simulator seeds trip requests without any UI interaction.

## License

Apache 2.0.
