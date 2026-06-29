# Akka Sample: Surprise Trip Planner

A supervisor agent delegates destination research, logistics, and activities to worker agents, then assembles a surprise travel itinerary from a traveler's preferences.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One AI key option (the generator asks which you want when none is set): a mock provider (no key), an existing env var (`ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`), an env-file path, a secrets-store URI, or type-once in the session.
- Host software: None. This blueprint runs out of the box — every external system is modeled inside the same service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.planning-travel.surprise-trip-planner ~/my-projects/surprise-trip-planner
cd ~/my-projects/surprise-trip-planner
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or preference defaults.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. Section 12 of `SPEC.md` instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding.

## What you'll get

- Four agents: a `TripSupervisorAgent` that delegates, and three workers — `DestinationAgent`, `LogisticsAgent`, `ActivitiesAgent`.
- A `TripWorkflow` orchestrating delegation and assembly.
- A `TripEntity` (event-sourced) holding the trip lifecycle, plus a `TripsView` read model.
- A `TripRequestConsumer`, a `RequestSimulator` and a `StalledTripMonitor` (timed actions).
- Two HTTP endpoints (API + static UI) and a single self-contained UI with five tabs.

## Customise before generating

- `SPEC.md` Section 1 — the system name and one-line pitch.
- `SPEC.md` Section 11 — the model provider default and HTTP port.
- `prompts/` — the four agent prompts (tone, refusal patterns, output forms).

## What gets validated

- A submitted preference set produces a destination, logistics, and an activities plan that assemble into a `READY` itinerary.
- The before-tool-call guardrail blocks a worker's external search/booking tool when it is out of the granted scope.
- The before-agent-response guardrail blocks an itinerary whose travel facts (visa, price ranges) fail the grounding check.
- A trip left mid-plan past the stall threshold transitions to `ESCALATED`.

## License

Apache 2.0.
