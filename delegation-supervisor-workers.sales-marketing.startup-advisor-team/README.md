# Akka Sample: Startup Advisor (Multi-Agent)

A supervisor agent receives a startup profile and delegates work to four specialist agents running **in parallel**: market research, go-to-market strategy, content planning, and product roadmap. The supervisor merges their outputs into one coherent startup advisory report. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the startup profile stream and the specialist tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.sales-marketing.startup-advisor-team  ~/my-projects/startup-advisor
cd ~/my-projects/startup-advisor
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or advisory report schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **AdvisorSupervisor** — AutonomousAgent that decomposes a startup profile into work items and synthesises the specialist outputs into one advisory report.
- **MarketResearcher** — AutonomousAgent that maps the competitive landscape and target segments.
- **GtmStrategist** — AutonomousAgent that drafts go-to-market channels, pricing signals, and launch sequencing.
- **ContentPlanner** — AutonomousAgent that outlines a content and thought-leadership plan.
- **RoadmapAdvisor** — AutonomousAgent that proposes a phased product roadmap.
- **AdvisoryWorkflow** — Workflow that fans the work out to all four specialists in parallel, then calls AdvisorSupervisor for synthesis.
- **AdvisorySessionEntity** — EventSourcedEntity holding the full advisory-session lifecycle.
- **AdvisoryView** — projection the UI streams via SSE.
- **AdvisoryEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the startup profiles the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `AdvisoryReport` record fields (e.g., add `fundingReadinessScore`).
- `prompts/market-researcher.md` — narrow the agent to a specific vertical.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real market-data API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a startup profile → session enters `PLANNING`, then `IN_PROGRESS`, then `SYNTHESISED`.
2. Workers fail-fast → if any specialist times out, the session enters `DEGRADED` with whichever partial outputs arrived.
3. Eval-event sampling captures one synthesis decision and surfaces the score on the App UI.

## License

Apache 2.0.
