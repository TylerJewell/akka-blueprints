# Architecture

This file is the narrative around the four diagrams in [`../PLAN.md`](../PLAN.md). The generated system renders those diagrams on the Architecture tab; this prose is what an author reads to understand why each component exists.

## The turn-taking loop

The system has one job: run a moderated negotiation. A request arrives — an item, a buyer budget, a seller floor — and the `NegotiationWorkflow` becomes the Facilitator. It does not negotiate itself; it owns the turn order. Each round it gives the Buyer one turn, then the Seller one turn, then asks the Facilitator agent to judge the round. That judgement routes the workflow: a deal ends the loop with a final price, a breakdown ends it with no deal, and anything else advances the round. Ten rounds is the hard cap; the eleventh would conclude as no deal.

This is the moderation-turn-taking pattern in its plainest form. The two parties never call each other. The moderator is the only component that talks to both, and the moderator alone decides when a round is over and when the whole negotiation is over. Keeping the turn order inside a durable workflow means a crash mid-round resumes at the same round with the same offer history, because the round counter and every offer live on the `NegotiationEntity`, not only in workflow memory.

See the **component graph** (PLAN.md §1) for who calls whom. Solid arrows are commands, dashed arrows are event subscriptions, dotted arrows are the two scheduled ticks.

## How a negotiation flows

The **interaction sequence** (PLAN.md §2) traces one negotiation from request to score. A request lands in the `InboundRequestQueue` — either from the App UI tab or from the `RequestSimulator` that drips a canned scenario every thirty seconds. The `NegotiationRequestConsumer` picks up the queued event, checks the operator halt flag on `SystemControl`, and if the system is running it starts a workflow with a fresh id.

Inside the loop, each agent turn is a workflow step that calls `forAutonomousAgent(...).runSingleTask(...)` and then `forTask(taskId).result(...)` to retrieve the typed `Offer` or `FacilitatorDecision`. Because those calls hit an LLM, each step overrides the default five-second timeout to sixty seconds — the single most common cause of a turn-taking loop that "retries forever" is leaving the default in place. After the Facilitator concludes, the entity emits `NegotiationConcluded`, and the `OutcomeEvaluator` consumer scores the result and writes it back. The score is the last thing to appear beside the negotiation in the UI.

## The negotiation lifecycle

The **state machine** (PLAN.md §3) shows the `NegotiationEntity` moving `CREATED → NEGOTIATING → CONCLUDED`, with a side exit to `ESCALATED` when the `StalledNegotiationMonitor` finds a negotiation still running after two minutes. The deal-or-no-deal distinction is not in the enum — it rides on a separate `outcome` string — so the read model never has to index an enum column, which the runtime cannot auto-index. `OutcomeEvaluated` is a self-transition on `CONCLUDED`: it enriches the row without changing the lifecycle stage.

## What is stored

The **entity model** (PLAN.md §4) shows the `Negotiation` aggregate with its append-only `OfferLine` history, the events it emits, and the one-to-one projection into the view row the UI reads. The `InboundRequestQueue` and `SystemControl` are the two supporting pieces of durable state: one records incoming requests, the other holds the halt flag. Everything is in one service — the market, the parties, the moderator, and the operator switch are all Akka primitives, so the whole system builds and runs with `/akka:build` and nothing else.

## Why these primitives

- **Three AutonomousAgents, not three request/response Agents.** Each party runs a short bounded task per turn and returns a typed result; the runtime drives the task to completion. The Buyer and Seller are symmetric; the Facilitator is the neutral third.
- **A Workflow for the moderator.** Turn order is durable, restartable state. A plain loop in an endpoint would lose its place on a crash; the workflow resumes mid-negotiation.
- **An EventSourcedEntity for the negotiation.** Every offer is history. Replaying the events reconstructs the exact sequence of moves, which is what an audit of a concluded deal needs.
- **A KeyValueEntity for the halt switch.** The flag is a single mutable value with no history worth keeping; a key-value entity is the right fit.
- **Two Consumers.** One turns queued requests into workflows; one turns concluded negotiations into scored outcomes. Each reacts to events without the producer knowing it exists.
- **Two TimedActions.** One supplies load so the system is never idle; one enforces the two-minute escalation objective.
