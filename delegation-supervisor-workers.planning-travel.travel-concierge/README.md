# Akka Sample: Travel Concierge

A travel coordinator delegates destination research to a Destination Scout and pricing to a Fare Agent in parallel, then assembles a personalised itinerary. A before-tool-call guardrail gates every booking action, and a human-in-the-loop confirmation step holds the workflow before any purchase is committed. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound trip-request stream and the booking tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.planning-travel.travel-concierge  ~/my-projects/travel-concierge
cd ~/my-projects/travel-concierge
```

(Optional) Edit `SPEC.md` to change the destination seeds, model provider, or itinerary schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TripCoordinator** — AutonomousAgent that decomposes a trip request into a destination brief and a fare query, then assembles the final itinerary.
- **DestinationScout** — AutonomousAgent that researches destinations: attractions, advisories, visa requirements.
- **FareAgent** — AutonomousAgent that estimates flight and accommodation pricing.
- **TripWorkflow** — Workflow that fans the work out to DestinationScout and FareAgent in parallel, then calls TripCoordinator to assemble the itinerary, then holds for human confirmation before any booking tool call.
- **TripEntity** — EventSourcedEntity holding the full trip lifecycle.
- **TripView** — projection the UI streams via SSE.
- **TripEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the destination seeds the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `TripItinerary` record fields (e.g., add `carbonFootprint`).
- `prompts/destination-scout.md` — narrow the scout to a specific region.
- `eval-matrix.yaml` — tighten the guardrail hook or add a second HITL tier.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a trip request → trip enters `PLANNING`, then `RESEARCHING`, then `AWAITING_APPROVAL`, then `CONFIRMED`.
2. User rejects the itinerary at the HITL step → trip enters `DECLINED`; no booking tool call is issued.
3. A before-tool-call guardrail blocks a booking attempt when constraints are violated → trip enters `BLOCKED`.
4. A worker timeout degrades the itinerary → trip enters `DEGRADED` with partial results.

## License

Apache 2.0.
