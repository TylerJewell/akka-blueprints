# Akka Sample: Startup Advisor (MCP)

A startup advisor agent delegates go-to-market research to a Market Researcher and channel strategy work to a Growth Analyst, both running in parallel via MCP-exposed tools, then synthesises actionable GTM guidance. Demonstrates the **delegation-supervisor-workers** coordination pattern with an MCP tool allow-list guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the MCP tool surface and the advisory tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.sales-marketing.startup-advisor-mcp  ~/my-projects/startup-advisor-mcp
cd ~/my-projects/startup-advisor-mcp
```

(Optional) Edit `SPEC.md` to change the business scenarios the simulator drips, or remove the simulator entirely.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AdvisorCoordinator** — AutonomousAgent that decomposes a startup brief into a market research query and a growth channel question, then synthesises the workers' outputs into GTM guidance.
- **MarketResearcher** — AutonomousAgent that calls MCP-exposed market-data tools to return a `MarketSnapshot` (TAM, competitive landscape, buyer segments).
- **GrowthAnalyst** — AutonomousAgent that evaluates channel options via MCP-exposed channel-scoring tools and returns a `ChannelPlan`.
- **AdvisoryWorkflow** — Workflow that fans the work out to MarketResearcher and GrowthAnalyst in parallel, then asks AdvisorCoordinator to synthesise a `GtmGuidance` record.
- **AdvisorySessionEntity** — EventSourcedEntity holding the full session lifecycle.
- **AdvisoryView** — projection the UI streams via SSE.
- **AdvisoryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the business scenarios the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `GtmGuidance` record fields (e.g., add `confidenceScore`).
- `prompts/coordinator.md` — narrow the advisor to a specific industry vertical.
- `eval-matrix.yaml` — add further guardrail controls when you wire real external MCP tools.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a startup brief → session enters `SCOPING`, then `IN_PROGRESS`, then `ADVISED`.
2. Workers fail-fast → if either MarketResearcher or GrowthAnalyst times out, the session enters `DEGRADED` with whichever partial output exists.
3. The MCP allow-list guardrail blocks a disallowed tool call before it executes.
4. Wait after a successful synthesis → the session's row in the App UI shows an eval score.

## License

Apache 2.0.
