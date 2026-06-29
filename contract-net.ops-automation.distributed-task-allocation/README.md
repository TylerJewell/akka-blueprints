# Akka Sample: Contract-Net Task Auctioneer

This sample demonstrates the Contract-Net coordination pattern: a manager entity announces tasks to a pool of contractor agents, collects bids, validates the winning bid before award, assigns the task, monitors execution, and triggers renegotiation on failure. An LLM-backed bid evaluator ranks incoming bids against task requirements; a second LLM agent scores contractor performance after completion, feeding a leaderboard that throttles future awards to poor performers. The pattern covers the full lifecycle from announcement through renegotiation, with two observable governance controls wired inline.

## Prerequisites

- **Claude Code** with the Akka plugin installed (`/akka:setup` if not yet done)
- **Java 21+** and **Maven 3.9+** on PATH
- **Akka CLI** authenticated (`akka auth login`)
- An LLM API key — choose one approach:

  | Option | How |
  |--------|-----|
  | Mock LLM (no key needed) | Set `MOCK_LLM=true` in `application.conf` dev overrides |
  | Environment variable | `export ANTHROPIC_API_KEY=<your-key>` before starting |
  | `.env` file | Create `.env` at project root with `ANTHROPIC_API_KEY=<your-key>` (gitignored) |
  | Akka secrets URI | `akka secret create anthropic-key --value <your-key>` then reference `akka://secrets/anthropic-key` in config |
  | Type-once at startup | The service prompts on first request when no key is configured |

> **Never write your API key value into any tracked file.**

## Quick start

```bash
# 1. Generate the implementation from this spec
/akka:specify          # reads SPEC.md, produces code scaffold

# 2. Build and test locally
/akka:build

# 3. Run locally (port 9202)
/akka:deploy --local

# 4. Open the UI
open http://localhost:9202
```

## What you will see

1. **Auction board** — live SSE feed of open task announcements with bid counts updating in real time.
2. **Bid submission** — POST a bid for a task; the `BidEvaluatorAgent` ranks it against the spec and incumbent bids within seconds.
3. **Award flow** — once the bidding window closes the guardrail fires, the winning bid is validated, and the task moves to `IN_PROGRESS`.
4. **Completion & scoring** — mark a task complete; the `PerformanceScorerAgent` evaluates the outcome and updates the contractor's leaderboard score.
5. **Renegotiation** — mark a task failed; `RenegotiationWorkflow` reopens bidding, excluding the failed contractor.
6. **Contractor leaderboard** — SSE-streamed ranking that reflects live performance scores; contractors below threshold are flagged and throttled.
