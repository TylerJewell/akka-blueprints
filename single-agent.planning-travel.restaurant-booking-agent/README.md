# Akka Sample: SK Restaurant Booking

A single restaurant-booking agent collects reservation parameters from the user, validates them against availability rules, asks the user for confirmation, and then writes the booking to the Microsoft Graph Bookings API. The write is guarded by a `before-tool-call` guardrail so no external call goes out without a valid, user-confirmed payload.

Demonstrates the **single-agent** coordination pattern wired with two governance mechanisms: a `before-tool-call` guardrail that validates every booking write before the API call fires, and a human-in-the-loop confirmation step that pauses execution and presents a summary to the user before the reservation is committed.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the Microsoft Graph Bookings integration is simulated in-process and the agent's tool calls are handled by a local stub.

## Generate the system

```sh
cp -r ./single-agent.planning-travel.restaurant-booking-agent  ~/my-projects/restaurant-booking
cd ~/my-projects/restaurant-booking
```

(Optional) Edit `SPEC.md` to point at a different restaurant catalogue or to add additional availability constraints (e.g., restrict allowed party sizes or supported time slots).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BookingAgent** — an AutonomousAgent that collects restaurant, date, time, party size, and contact details from the user, asks for confirmation, then calls the booking tool.
- **BookingWorkflow** — orchestrates the collect → confirm → commit lifecycle per booking request.
- **BookingEntity** — an EventSourcedEntity holding the per-booking lifecycle from INITIATED through CONFIRMED.
- **ConfirmationConsumer** — a Consumer that listens for `ConfirmationRequested` events and routes the confirmation prompt to the active workflow.
- **BookingView + BookingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — replace the seeded restaurant catalogue (JSON under `src/main/resources/sample-events/restaurants.json` after generation) with your own venue list.
- `SPEC.md §5` — extend `BookingRequest` with additional fields (e.g., dietary requirements, occasion, preferred seating).
- `prompts/booking-agent.md` — constrain the agent's scope (a corporate travel deployer would restrict it to approved venues; a healthcare deployer might add accessibility fields).
- `eval-matrix.yaml` — wire a real Microsoft Graph Bookings client by swapping the stub implementation named under the guardrail mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user requests a table → the agent collects parameters → presents a confirmation summary → the user approves → the booking is committed and a confirmation number appears.
2. The agent attempts to call the booking tool with an invalid payload (past date, zero party size) → the `before-tool-call` guardrail rejects it → the agent corrects the parameters and retries.
3. The user declines the confirmation → the booking is cancelled with no external write; the entity records `BookingDeclined`.
4. A booking committed to a stub that returns a conflict error transitions the entity to `FAILED` with a clear reason visible in the UI.

## License

Apache 2.0.
