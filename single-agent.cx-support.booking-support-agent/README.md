# Akka Sample: Customer Support Booking Agent

A single customer-support agent answers natural-language questions about travel bookings, looks up reservation details, applies itinerary changes, and cancels bookings on behalf of the customer. Destructive operations (modify, cancel) pass through a `before-tool-call` guardrail that requires the booking to belong to the requesting customer before the tool executes.

Demonstrates the **single-agent** coordination pattern in the cx-support domain. Three governance mechanisms surround the agent: a PII sanitizer that scrubs customer and booking records before they enter the conversation context, a `before-tool-call` guardrail that validates destructive operations against ownership and business rules, and a CI gate backed by LLM-judge test assertions that run on every pull request.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The booking corpus lives in-process; the agent's tool calls are simulated against a seeded in-memory dataset.

## Generate the system

```sh
cp -r ./single-agent.cx-support.booking-support-agent  ~/my-projects/booking-support-agent
cd ~/my-projects/booking-support-agent
```

(Optional) Edit `SPEC.md` to add booking channels (hotel, rail, ferry) beyond the seeded air-travel examples, or to restrict which actions are permitted without a human override.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BookingSupportAgent** — an AutonomousAgent that accepts a customer message, optionally a session history, and the requesting customer id; returns a typed `AgentReply` carrying a natural-language response and any booking action taken.
- **BookingSessionWorkflow** — orchestrates the per-session turn loop: sanitize → agent call → record turn.
- **BookingSessionEntity** — an EventSourcedEntity holding the per-session conversation state, tool-call log, and outcome.
- **PiiSanitizer** — a Consumer that subscribes to `CustomerMessageReceived` events and redacts PII from the message text before the agent call.
- **BookingSessionView + SupportEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — extend the seeded booking dataset (flights, seat maps, fare rules) in `src/main/resources/sample-events/bookings.jsonl`.
- `SPEC.md §5` — add booking-channel-specific fields to `BookingRecord` (e.g., `trainNumber`, `cabinClass`, `hotelConfirmationCode`).
- `prompts/booking-support-agent.md` — narrow the agent's permitted actions (e.g., disable cancellations entirely and require agent escalation for refunds).
- `eval-matrix.yaml` — point the before-tool-call guardrail at your company's actual ownership-verification service instead of the in-process check.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer asks "What is my flight time?" → the agent looks up the booking and replies with departure and arrival details.
2. A customer asks to cancel a booking they own → the guardrail passes → the booking is cancelled → the reply confirms cancellation.
3. A customer asks to cancel a booking that belongs to a different customer → the guardrail blocks the tool call → the agent replies that it cannot process the request.
4. A customer message containing a credit card number is submitted → the LLM call log shows only `[REDACTED-PAYMENT-CARD]`; the entity's raw message is retained for audit.
5. The LLM-judge CI gate rejects a generated reply that hallucinates flight details not present in the booking record.

## License

Apache 2.0.
