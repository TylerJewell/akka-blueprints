# Architecture — sales-offer-generator

Four mermaid diagrams describe the system. The full source for each lives in `PLAN.md`; this file gives the narrative around them.

## Component graph

A brief enters through `OfferEndpoint` (or the `RequestSimulator` timer) and is recorded on `InboundRequestQueue`. `RequestConsumer` subscribes to that queue, runs `PiiSanitizer` over the brief, and starts one `OfferWorkflow` per brief with a fresh UUID. The workflow drives four `AutonomousAgent`s in sequence — `NeedsAnalyst`, `ProductMatcher`, `PricingStrategist`, `OfferComposer` — each consuming the prior agent's typed output. `ProductMatcher` reaches the in-process `CatalogEndpoint` through a catalog tool. Every stage writes an event to `OfferEntity`, which projects into `OffersView`. The UI reads the list and a live SSE stream from `OfferEndpoint`. `AppEndpoint` serves the static UI.

## Interaction sequence

The sequence diagram traces one brief end to end: enqueue, sanitize, then analyze → match → price → compose, with an entity write after each agent. The before-agent-response guardrail runs `PricingPolicy` inside the compose stage; `reviewStep` re-runs the same check as the durable gate and transitions the offer to `APPROVED` or `REJECTED`.

## State machine

`OfferEntity` moves through `QUEUED → ANALYZED → MATCHED → PRICED → COMPOSED`, then forks to `APPROVED` (policy passes) or `REJECTED` (policy fails). Both are terminal. State-label and edge-label colours rely on the Lesson 24 CSS overrides in `index.html`, not theme variables alone.

## Entity model

`InboundRequestQueue` spawns many `Offer` rows; each `Offer` projects one-to-one into the `OffersView` row. The view row carries the id, status, and quoted total for the list UI; the full record carries the structured needs, products, pricing, and offer body.
