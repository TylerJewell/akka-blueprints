# Akka Sample: HITL Order Processing

An order is submitted by a client, an agent validates and prices it, the workflow pauses at a human approval gate, and a fulfillment agent dispatches it only after a person approves it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.ops-automation.order-processing-hitl  ~/my-projects/order-processing
cd ~/my-projects/order-processing
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ValidationAgent` (AutonomousAgent) — validates the order and computes a pricing estimate; returns a typed `OrderReview{lineItems, estimatedTotal, riskFlags}`.
- `FulfillmentAgent` (AutonomousAgent) — dispatches an approved order to the simulated fulfillment target; returns a typed `DispatchConfirmation{trackingId, dispatchedAt}`.
- `OrderWorkflow` (Workflow) — a 3-task graph: validate → await approval → fulfill.
- `OrderEntity` (EventSourcedEntity) — the order lifecycle and its events.
- `OrdersView` (View) — a read model the UI queries and streams over SSE.
- `OrderEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/validation-agent.md` — the validation and pricing instructions.

## What gets validated

- Submitting an order triggers validation that appears in `PENDING_APPROVAL` with a non-empty review summary.
- Approving a `PENDING_APPROVAL` order drives it to `FULFILLED` with a non-null tracking id.
- Rejecting a `PENDING_APPROVAL` order drives it to terminal `REJECTED` with the reason shown.
- The fulfillment step never runs unless the order is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
