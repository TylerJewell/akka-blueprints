# Akka Sample: Trip Planner Multi-Agent

A trip-planning supervisor delegates destination research, itinerary drafting, and booking tasks to three specialist agents running in parallel, then compiles a confirmed travel plan. Demonstrates the **delegation-supervisor-workers** coordination pattern with a booking guardrail and human-in-the-loop approval before any reservation is made.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the destination data, itinerary logic, and booking tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.planning-travel.trip-planner-team-mac  ~/my-projects/trip-planner
cd ~/my-projects/trip-planner
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or destination schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **TripSupervisor** — AutonomousAgent that decomposes a trip request into parallel work items and compiles the final plan.
- **DestinationResearcher** — AutonomousAgent that gathers destination facts (climate, visa requirements, highlights).
- **ItineraryPlanner** — AutonomousAgent that drafts a day-by-day itinerary from destination data.
- **BookingAgent** — AutonomousAgent that constructs reservation payloads for flights and accommodation; never executes without guardrail clearance and user approval.
- **TripWorkflow** — Workflow that fans work out to the three specialists in parallel, gates on the booking guardrail, then waits for the human-in-the-loop approval signal.
- **TripEntity** — EventSourcedEntity holding the full trip lifecycle.
- **ApprovalEntity** — EventSourcedEntity tracking the HITL approval token.
- **TripView** — projection the UI streams via SSE.
- **TripEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the destination presets the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `TripPlan` record fields (e.g., add `budgetEstimateUsd`).
- `prompts/destination-researcher.md` — narrow the agent to a specific region.
- `eval-matrix.yaml` — tighten the guardrail flavor or swap the HITL trigger condition.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a trip request → plan enters `PLANNING`, then `IN_PROGRESS`, then `AWAITING_APPROVAL` with a full draft; user approves → `CONFIRMED`.
2. Booking guardrail blocks a plan that exceeds the stated budget ceiling → plan enters `BLOCKED`.
3. User rejects the draft → plan enters `REJECTED`; no booking is made.
4. Wait for the background simulator to drip a request → the App UI is non-empty without user interaction.

## License

Apache 2.0.
