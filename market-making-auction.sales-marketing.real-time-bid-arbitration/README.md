# Akka Sample: Real-Time Bid Arbitration Agent

An agent receives auction impression requests and decides whether to bid and at what price. Budget enforcement halts bidding when a per-window spend cap is reached. A periodic sampler tracks win-rate and ROAS against the campaign envelope. Anomalous bid bursts escalate to an ops queue for human review. Demonstrates the **market-making-auction** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the auction request stream and the bidding exchange are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./market-making-auction.sales-marketing.real-time-bid-arbitration  ~/my-projects/real-time-bid-arbitration
cd ~/my-projects/real-time-bid-arbitration
```

(Optional) Edit `SPEC.md` to change the campaign parameters, model provider, or bid-price schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **BidArbitrationAgent** — AutonomousAgent that evaluates each auction impression and returns a `BidDecision` (bid/no-bid + price + reasoning).
- **BidWorkflow** — Workflow that drives the per-slot lifecycle: evaluate → place bid → record result → apply budget charge.
- **CampaignBudgetEntity** — EventSourcedEntity tracking cumulative spend per window; emits `CapBreached` to halt further bids.
- **AuctionSlotEntity** — EventSourcedEntity holding each slot's lifecycle state from `RECEIVED` through `WON`, `LOST`, `SKIPPED`, `HALTED`, or `ESCALATED`.
- **AuctionRequestQueue** — EventSourcedEntity that ingests incoming auction requests for fan-out via Consumer.
- **AuctionRequestConsumer** — Consumer that starts one `BidWorkflow` per submitted slot.
- **BidLedgerView** — View projection the UI streams via SSE.
- **AuctionSimulator** — TimedAction dripping a sample auction request every 30 seconds.
- **PerformanceSampler** — TimedAction evaluating win-rate and ROAS every 5 minutes and emitting a `PerformanceEvalRecorded` event.
- **BidEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change campaign budget, window duration, burst threshold, or remove the simulator.
- `SPEC.md §5` — adjust `AuctionSlot` record fields (e.g., add `targetCpm`).
- `prompts/bid-arbitration-agent.md` — narrow the agent to a specific channel (display, video, search).
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real exchange API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an auction request → slot progresses `RECEIVED → EVALUATING → BID_PLACED → WON/LOST` within 30 s; UI reflects each transition via SSE.
2. Budget cap reached → subsequent slots enter `HALTED`; the campaign's `CapBreached` event is visible in the ledger.
3. Anomalous burst detected → affected slots enter `ESCALATED`; ops queue receives the alert.
4. Wait for `PerformanceSampler` → slots gain a `PerformanceEvalRecorded` annotation visible in the App UI.

## License

Apache 2.0.
