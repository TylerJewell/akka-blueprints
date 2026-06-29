# Akka Sample: Travel Support Router

A `RouterAgent` classifies an incoming travel-support request and hands the same `RESOLVE` task off to a `FlightSpecialist`, `HotelSpecialist`, `CarRentalSpecialist`, or `ExcursionSpecialist` that owns the resolution end-to-end. Demonstrates the **handoff-routing** coordination pattern wired with three governance mechanisms: a PII sanitizer before any LLM call, a before-tool-call guardrail that checks booking/cancellation mutations against policy before execution, and application-level HITL that pauses sensitive booking changes for explicit passenger confirmation.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- Host-software requirement: **None** — this blueprint runs out of the box. The inbound request stream and the outbound resolution surface are both modeled inside the service.

## Generate the system

```sh
cp -r ./handoff-routing.cx-support.travel-support-router  ~/my-projects/travel-support-router
cd ~/my-projects/travel-support-router
```

(Optional) Edit `SPEC.md` to point `RequestSimulator` at a real travel CRM source or to change the routing taxonomy (add `LOUNGE_ACCESS`, `VISA_ASSISTANCE`, etc.).

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. `SPEC.md` Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **RequestSimulator** — TimedAction firing every 30 s that drips canned travel-support requests from a JSONL file into `RequestQueue`.
- **RequestQueue** — EventSourcedEntity append-only log of every inbound request (audit before redaction).
- **PiiSanitizer** — Consumer that redacts passenger names, passport numbers, loyalty numbers, and payment card digits before any LLM call.
- **RouterAgent** — typed Agent that classifies the sanitized request into `FLIGHTS`, `HOTELS`, `CAR_RENTAL`, `EXCURSIONS`, or `UNCLEAR`.
- **FlightSpecialist** — AutonomousAgent that owns the `RESOLVE` task for flight-related requests (rebooking, seat changes, upgrades, cancellations).
- **HotelSpecialist** — AutonomousAgent that owns the `RESOLVE` task for hotel-related requests (check-in/out changes, room type, amenity questions).
- **CarRentalSpecialist** — AutonomousAgent that owns the `RESOLVE` task for car rental requests (extension, upgrade, damage waiver questions).
- **ExcursionSpecialist** — AutonomousAgent that owns the `RESOLVE` task for excursion and activity requests (availability, cancellation, rebooking).
- **BookingGuardrail** — typed Agent that checks proposed booking/cancellation tool calls against the travel-policy rubric before execution.
- **ConfirmationGate** — typed Agent that emits a HITL pause, presents a plain-language summary of the proposed change to the passenger, and resumes only on `CONFIRM` or `REJECT`.
- **RouterEvalScorer** — Consumer that listens for `RoutingDecided` events and writes an inline routing-quality score.
- **TravelWorkflow** — Workflow per request: sanitize → route → specialize → guardrail → confirm → resolve → publish.
- **RequestEntity** — EventSourcedEntity holding each request's lifecycle.
- **RequestView + TravelEndpoint + AppEndpoint** — read model + REST/SSE + static UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the routing taxonomy (add `LOUNGE_ACCESS`, `TRAVEL_INSURANCE`, etc.) and add the matching specialist agent.
- `SPEC.md §5` — extend the `TravelRequest` record with deployer-specific fields (`loyaltyTier`, `bookingReference`, `originChannel`).
- `prompts/router-agent.md` — narrow the classifier rules (category-specific keywords, default-to-escalation thresholds).
- `prompts/flight-specialist.md` / `prompts/hotel-specialist.md` / etc. — encode brand voice, change-fee authority limits, escalation triggers.
- `eval-matrix.yaml` — swap the in-process PII regex for a real redactor or adjust the HITL confirmation threshold.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Simulator drips a flight-change request → sanitized, routed to `FLIGHTS`, handed to `FlightSpecialist`, guardrail clears, confirmation gate gets a `CONFIRM`, and the resolution is published.
2. Simulator drips a hotel request → routed to `HOTELS` → resolved by `HotelSpecialist`.
3. An ambiguous request routes `UNCLEAR` and lands in `ESCALATED` without any specialist invocation.
4. A specialist tool call that violates the before-tool-call guardrail (e.g. initiating a non-refundable cancellation outside policy) is blocked; the request lands in `BLOCKED`.
5. A passenger `REJECT` at the confirmation gate terminates the workflow in `PASSENGER_REJECTED` without any booking mutation.
6. A routing score (1–5) appears on every routed request within ~10 s of the routing decision.

## License

Apache 2.0.
