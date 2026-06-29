# Akka Sample: Customer Service ReAct Agent

A single ReAct agent handles airline customer service requests end-to-end: it reasons over a set of tools (search bookings, modify a reservation, file a complaint) and loops until the task is resolved or its iteration budget is exhausted. Every booking modification goes through a human-in-the-loop confirmation gate before the write executes; PII is stripped from customer input before the agent sees it; and a `before-tool-call` guardrail blocks any destructive reservation write that arrives without a prior confirmation token.

Demonstrates the **single-agent** coordination pattern wired with three governance mechanisms: a PII sanitizer that runs before the agent receives the customer message, a `before-tool-call` guardrail that enforces the confirmation-token protocol on destructive writes, and an application-level human-in-the-loop gate that pauses execution until the customer confirms the intended change.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. Booking data lives in-process; the agent's tool calls are simulated against a seeded in-memory record store.

## Generate the system

```sh
cp -r ./single-agent.cx-support.airline-cs  ~/my-projects/airline-cs
cd ~/my-projects/airline-cs
```

(Optional) Edit `SPEC.md` to swap the seeded booking records for your own test data, or extend the complaint categories in Section 5.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CustomerServiceAgent** — an AutonomousAgent that accepts a customer message and booking context, reasons over tools in a ReAct loop, and returns a typed `ServiceOutcome`.
- **ServiceWorkflow** — orchestrates sanitize-wait → serve → hitl-confirm (when needed) → complete, per incoming request.
- **RequestEntity** — an EventSourcedEntity holding the per-request lifecycle.
- **MessageSanitizer** — a Consumer that subscribes to `RequestReceived` events, redacts PII from the customer message, and emits `MessageSanitized` back to the entity.
- **HitlGateway** — a Workflow step that pauses execution and issues a confirmation prompt to the customer; resumes when the customer submits a `confirm` or `cancel` token.
- **RequestView + ServiceEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI matching the formal exemplar: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — extend the seeded booking records in `src/main/resources/sample-events/bookings.jsonl` to match your test scenarios.
- `SPEC.md §5` — add airline-specific fields to `ServiceOutcome` (e.g., `loyaltyTier`, `fareClass`, `refundEligibility`).
- `prompts/customer-service-agent.md` — tighten the agent's tool-use policy (e.g., restrict `modifyReservation` to same-day changes only, or require a specific fare class before waiving fees).
- `eval-matrix.yaml` — wire a real PII redactor in the sanitizer mechanism's implementation paragraph if you need production-grade redaction.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A customer asks to change their seat → the agent calls `searchBooking`, presents options, issues a HITL confirmation prompt, and on customer approval calls `modifyReservation`.
2. The agent attempts a reservation modification without a valid confirmation token → the `before-tool-call` guardrail blocks the call → the agent re-requests confirmation → the second attempt carries the token and succeeds.
3. A customer message containing PII (email, frequent-flyer number) is sanitized before the agent receives it; the raw form is preserved on the entity for audit.
4. A customer files a complaint → the agent calls `fileComplaint`, receives a case number, and returns it in the `ServiceOutcome`.

## License

Apache 2.0.
