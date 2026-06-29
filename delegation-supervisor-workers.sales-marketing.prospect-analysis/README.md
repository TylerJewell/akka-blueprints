# Akka Sample: Prospect Analysis Team

A supervisor workflow delegates to three worker agents that research a target company, analyze its org structure to surface decision-makers, and draft an outreach strategy.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One AI key option (the generator asks how to source it if none is set): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, `GOOGLE_AI_GEMINI_API_KEY`, or pick the mock provider for offline runs.
- Host software: none. This blueprint runs out of the box — the web research surface is modeled inside the service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.sales-marketing.prospect-analysis ~/my-projects/prospect-analysis
cd ~/my-projects/prospect-analysis
```

(Optional) Edit `SPEC.md` to rename the system, change the model provider, or adjust the allowed research domains.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding.

## What you'll get

- Three worker agents: `CompanyResearchAgent`, `OrgAnalysisAgent`, `OutreachStrategyAgent` (each an AutonomousAgent).
- `ProspectWorkflow` — the supervisor that delegates each phase to a worker and records the result.
- `ProspectEntity` (event-sourced) and `RequestQueueEntity` for inbound work.
- `ProspectsView` — CQRS read model the UI queries and streams over SSE.
- `AnalysisRequestConsumer` starting one workflow per queued company.
- `RequestSimulator` and `StalledAnalysisMonitor` TimedActions.
- Two HttpEndpoints plus a single self-contained UI.

## Customise before generating

- System name and short name — SPEC.md Section 1 and the UI title.
- Model provider — SPEC.md Section 11 identity block.
- Allowed research domains — the guardrail allow-list in `eval-matrix.yaml` control G1 and `prompts/company-research-agent.md`.

## What gets validated

- Submit a company and watch the supervisor drive it through research, analysis, and strategy (`reference/user-journeys.md` J1).
- A blocked off-domain research call is refused by the before-tool-call guardrail (J2).
- Decision-maker contact details are masked before they reach the read model (J3).
- A stalled analysis is marked automatically by the periodic monitor (J4).

## License

Apache 2.0.
