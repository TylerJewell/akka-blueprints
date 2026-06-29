# Akka Sample: Contract-Net Satellite Tasker

An auction-based task planner that distributes observation requests across a satellite fleet. Each observation window is auctioned to the satellite that submits the winning bid; a bid feasibility guardrail blocks bids that contradict the satellite's ephemeris before acceptance. Demonstrates the **contract-net** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the observation-request stream and the satellite bid tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./contract-net.research-intel.satellite-task-planner  ~/my-projects/satellite-task-planner
cd ~/my-projects/satellite-task-planner
```

(Optional) Edit `SPEC.md` to change the satellite fleet size, bid scoring weights, or observation-window duration.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AuctionCoordinator** — AutonomousAgent that opens an auction for an observation request, scores bids, and selects the winner.
- **SatelliteBidder** — AutonomousAgent that evaluates an observation request against a satellite's current ephemeris window and returns a bid.
- **ObservationAuction** — Workflow that broadcasts the request to all eligible satellites, collects bids within a deadline, runs the feasibility guardrail on each bid, and awards the task to the winner.
- **ObservationRequestEntity** — EventSourcedEntity holding the full observation-request lifecycle.
- **SatelliteEntity** — EventSourcedEntity holding each satellite's operational state.
- **ObservationView** — projection the UI streams via SSE.
- **ObservationEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the fleet size or the observation-window parameters the simulator uses.
- `SPEC.md §5` — adjust the `ObservationRequest` record fields (e.g., add `priorityLevel` or `targetRegion`).
- `prompts/satellite-bidder.md` — constrain bidding to a specific orbit type.
- `eval-matrix.yaml` — tighten the scheduled-eval cadence if you need faster completion-rate feedback.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit an observation request → auction opens, bids arrive, winner is awarded, request enters `AWARDED`.
2. Infeasible bid submitted → feasibility guardrail blocks the bid before the coordinator evaluates it; auction continues with remaining bids.
3. No bids received within the deadline → request enters `UNALLOCATED`.
4. Awarded task completes → completion-rate eval scores the fleet's awarded-task throughput.

## License

Apache 2.0.
