# Akka Sample: Flight Booking

A `BookingAgent` walks a traveller's request through three task phases — **SEARCH → SELECT → BOOK** — wired together by explicit task dependencies. The SEARCH phase finds available flights, the SELECT phase picks the best seat on the chosen flight, and the BOOK phase commits the reservation — but only after the user confirms the itinerary. A `before-tool-call` guardrail blocks the irreversible booking write until user confirmation is recorded; if the user has not confirmed, the tool call is rejected before any money moves.

Demonstrates the **sequential-pipeline** coordination pattern in a travel-booking domain, with two governance mechanisms: a `before-tool-call` guardrail that blocks the booking write until explicit user approval is on record, and an application-level human-in-the-loop confirmation gate that prevents commitment to an irreversible transaction.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Every flight search, seat inventory, and booking confirmation tool is implemented in-process inside the same Akka service using deterministic sample data.

## Generate the system

```sh
cp -r ./sequential-pipeline.planning-travel.flight-booking-pipeline  ~/my-projects/flight-booking
cd ~/my-projects/flight-booking
```

(Optional) Edit `SPEC.md` to point at a different set of seeded routes, a different model provider, or richer fare data.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BookingAgent** — one AutonomousAgent declaring three Task constants (`SEARCH_FLIGHTS`, `SELECT_SEAT`, `COMMIT_BOOKING`); the workflow runs them in order, feeding each typed output forward as the next task's instruction context.
- **BookingPipelineWorkflow** — runs `searchStep → selectStep → confirmStep → bookStep`. Each step calls `runSingleTask`, writes the typed result back onto `BookingEntity`, and pauses at `confirmStep` to wait for user approval before `bookStep` proceeds.
- **BookingEntity** — an EventSourcedEntity holding the per-booking lifecycle (`FlightsFound`, `SeatSelected`, `BookingConfirmed`, `BookingCommitted`, `BookingFailed`).
- **SearchTools / SelectTools / BookTools** — three function-tool classes registered on the agent, one per phase. The `before-tool-call` guardrail enforces that each tool is only callable in its own phase and that the irreversible `commitReservation` tool is only callable after `BookingConfirmed` is on record.
- **BookingGuardrail** — the runtime check that backs both the phase-dependency contract and the user-confirmation precondition. A tool call referencing a phase whose precondition has not yet been recorded is rejected before the tool runs.
- **BookingView + BookingEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the seeded routes under `src/main/resources/sample-data/routes/*.json` to cover your demo audience's travel corridors.
- `SPEC.md §4` and `prompts/booking-agent.md` — narrow the agent's role (e.g., restrict to specific airline alliances, enforce a maximum fare threshold, add baggage-allowance constraints) by tightening the system prompt and adjusting the typed records (`FlightOffer`, `SeatMap`, `BookingConfirmation`).
- `SPEC.md §5` — extend typed outputs with additional fare fields. The phase-gating guardrail checks recorded-phase preconditions, not field shapes, so adding fields does not require guardrail edits.
- `eval-matrix.yaml` — wire a real confirmation flow (replace the mock confirmation step with a real notification → webhook callback loop) by editing the `H1` implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a route → `SEARCH` runs → `SELECT` runs → the UI shows the itinerary and a **Confirm booking** button → user clicks confirm → `BOOK` runs → a typed `BookingConfirmation` lands in the UI. Every transition is visible in real time.
2. The `commitReservation` tool is called before the user confirms (forced via the mock LLM) → the `before-tool-call` guardrail rejects the call → the workflow records the rejection event → the pipeline waits at the confirmation step → the itinerary is never booked without approval.
3. Each task receives only its own typed inputs; the SEARCH task does not see seat-selection logic, and the BOOK task does not see raw flight-search results — the workflow's task-chaining is the only path information travels between phases.
4. A confirmed booking is idempotent: re-submitting the same booking ID after `BookingCommitted` returns the existing confirmation rather than creating a duplicate reservation.

## License

Apache 2.0.
