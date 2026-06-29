# Akka Sample: Sales Offer Generator

Paste a customer brief. Four agents run in sequence and hand back a policy-checked offer document.

## Prerequisites

- Claude Code with the Akka plugin installed. Install docs: <https://doc.akka.io/>.
- One model-provider API key, sourced at generation time via one of five options (mock, env-var name, env-file path, secrets URI, or type-once) — see `SPEC.md` Section 11. You never write the key value to disk.
- Host software: None. This blueprint runs out of the box — the product catalog is modeled in-process with Akka components over canned data.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — system name, model provider, agent prompts.
3. In Claude Code, run:

```
/akka:specify @SPEC.md
```

`SPEC.md` Section 12 auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When the service is up, Claude prints the listening URL.

## What you'll get

- Four agents: `NeedsAnalyst`, `ProductMatcher`, `PricingStrategist`, `OfferComposer`.
- `OfferWorkflow` coordinating analyze → match → price → compose → review.
- `OfferEntity` (event-sourced offer lifecycle) and `InboundRequestQueue`.
- `OffersView` read model with SSE streaming to the UI.
- `RequestConsumer` (runs the PII sanitizer), `RequestSimulator` (drips canned briefs).
- `OfferEndpoint`, `CatalogEndpoint` (in-process product catalog), `AppEndpoint`.
- A single self-contained 5-tab UI at `src/main/resources/static-resources/index.html`.

## Customise before generating

- System name and pitch — `SPEC.md` Section 1.
- Model provider default — `SPEC.md` Section 11 and the generated `application.conf`.
- Agent behavior — `prompts/needs-analyst.md`, `prompts/product-matcher.md`, `prompts/pricing-strategist.md`, `prompts/offer-composer.md`.

## What gets validated

The journeys in `reference/user-journeys.md`:

- Submit a brief and watch an offer reach `APPROVED`.
- An offer with an over-ceiling discount transitions to `REJECTED`.
- A brief with an email and phone is redacted before any agent prompt; events and logs carry no raw PII.
- The simulator seeds a brief and a fresh pipeline runs to a terminal state.

## License

Apache 2.0.
