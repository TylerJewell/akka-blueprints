# Akka Sample: Custom Orchestration Airline Assistant

An `IntentRouter` classifies an inbound passenger request by intent (booking, change, baggage, status), then hands the same task off to the matching specialist agent that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with three governance mechanisms: a PII sanitizer (PNR and passenger data stripped before any LLM call), a before-tool-call guardrail (checked before destructive reservation operations), and a before-agent-response guardrail (checked on every customer-facing draft).

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound passenger-request stream and the outbound response surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.airline-triage-router  ~/my-projects/airline-triage-router
cd ~/my-projects/airline-triage-router
```

(Optional) Edit `SPEC.md` to point `RequestFeeder` at a real airline PSS/CRM source, or to change the intent taxonomy (add `LOYALTY`, `ACCESSIBILITY`, etc.) and the matching specialist agent.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RequestFeeder** — TimedAction firing every 30 s that drips canned passenger requests from a JSONL file into `PassengerRequestQueue`.
- **PassengerRequestQueue** — EventSourcedEntity append-only audit log of every inbound request (captured before redaction).
- **PnrSanitizer** — Consumer that strips PNR locators, passport/ID numbers, frequent-flyer numbers, and email addresses before any LLM call.
- **IntentRouter** — typed Agent that classifies the sanitized request into `BOOKING`, `CHANGE`, `BAGGAGE`, `STATUS`, or `UNCLEAR`.
- **BookingSpecialist** — AutonomousAgent that owns the `RESOLVE` task for new-booking and rebooking requests.
- **ChangeSpecialist** — AutonomousAgent that owns the `RESOLVE` task for seat changes, flight changes, and cancellations.
- **BaggageSpecialist** — AutonomousAgent that owns the `RESOLVE` task for baggage-claim, lost-luggage, and excess-fee disputes.
- **StatusSpecialist** — AutonomousAgent that owns the `RESOLVE` task for flight-status and itinerary-status inquiries.
- **RoutingJudge** — typed Agent used by `RoutingEvalScorer` to grade every intent-classification decision against a 1–5 rubric.
- **ToolCallGuardrail** — typed Agent that checks destructive tool calls (seat mutations, cancellations, refunds) before they fire.
- **ResponseGuardrail** — typed Agent that checks every specialist draft before it reaches the passenger.
- **FlightRequestWorkflow** — Workflow per request: sanitize-handoff → classify → route → resolve → guardrail → publish.
- **FlightRequestEntity** — EventSourcedEntity holding each request's lifecycle.
- **FlightRequestView + AirlineEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- **RoutingEvalScorer** — Consumer that listens for `IntentClassified` events and writes an inline eval score.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — add intent categories (`LOYALTY`, `ACCESSIBILITY`, `MEAL`) and the matching specialist agent.
- `SPEC.md §5` — extend the `FlightRequest` record with deployer-specific fields (`loyaltyTier`, `marketingConsent`, `slaDeadline`).
- `prompts/intent-router.md` — narrow the classification rules (direct-channel vs. airport check-in thresholds, codeshare edge cases).
- `prompts/booking-specialist.md`, `prompts/change-specialist.md`, etc. — encode airline brand voice, fare-rule authority limits, and escalation triggers.
- `eval-matrix.yaml` — swap the in-process PNR regex for a real airline DLP sidecar if your data governance requires it.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Feeder drips a booking-intent request → sanitized, classified `BOOKING`, handed to `BookingSpecialist`, guardrail passes, response published.
2. Feeder drips a baggage-claim request → classified `BAGGAGE`, handled by `BaggageSpecialist`, published.
3. An ambiguous message classifies as `UNCLEAR`, workflow terminates in `UNRESOLVED`, no specialist invoked.
4. A before-tool-call guardrail blocks a cancellation attempt on a non-refundable fare; the request lands in `BLOCKED`.
5. A specialist draft that echoes a literal PNR token is blocked by the before-agent-response guardrail.
6. Every classified request carries a `RoutingScore` (1–5) and rationale within a few seconds of the classification.

## License

Apache 2.0.
