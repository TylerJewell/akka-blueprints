# Akka Sample: Agent-to-Agent Commerce Settlement

Autonomous buyer agents discover service listings, submit bids through a brokered auction, and settle the winning transaction — with a precondition guardrail that gates payment and a human-in-the-loop path for disputed orders.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the service listings and buyer agents are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./market-making-auction.general.agent-to-agent-settlement  ~/my-projects/agent-to-agent-settlement
cd ~/my-projects/agent-to-agent-settlement
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or settlement record fields.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BuyerAgent** — AutonomousAgent that evaluates service listings and submits competitive bids.
- **SettlementAgent** — AutonomousAgent that checks settlement preconditions before authorising payment.
- **AuctionWorkflow** — Workflow that closes an auction, runs the precondition guardrail, and settles the winning order.
- **ServiceListingEntity** — EventSourcedEntity holding the listing lifecycle (open → closed → cancelled).
- **OrderEntity** — EventSourcedEntity holding the order lifecycle (pending → prechecked → settled / disputed / cancelled).
- **MarketplaceView** — projection the UI streams via SSE.
- **AuctionConsumer** — Consumer that listens to listing events and triggers buyer agents.
- **ListingSimulator** — TimedAction that drips sample listings every 60 seconds.
- **DisputeMonitor** — TimedAction that scans stalled orders every 5 minutes and escalates to the operator.
- **MarketplaceEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the listing categories the simulator drips, or remove the simulator.
- `SPEC.md §5` — adjust the `Order` record fields (e.g., add `escrowRef`).
- `prompts/buyer-agent.md` — constrain the buyer to a maximum bid ceiling.
- `eval-matrix.yaml` — add a second guardrail if you wire a real payment processor.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Publish a listing → BuyerAgent bids → auction closes → precondition check passes → order reaches `SETTLED`.
2. Precondition check fails → order reaches `CANCELLED`; buyer receives a failure reason.
3. An already-settled order is flagged as disputed by the buyer → `DisputeMonitor` escalates to the operator queue → operator resolves it.
4. `ListingSimulator` drips listings; App UI is non-empty on first load.

## License

Apache 2.0.
