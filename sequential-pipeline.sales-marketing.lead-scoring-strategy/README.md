# Akka Sample: Lead Scoring Strategy

A pipeline that reads a lead's form responses, researches the company and industry, scores the lead, and writes a tailored engagement strategy — one stage feeding the next.

## Prerequisites

- Claude Code with the Akka plugin installed ([install docs](https://doc.akka.io/)).
- One model-provider API key, sourced at generation time (see the five options in `SPEC.md` Section 11). You can also pick a mock provider and run with no key.
- Host software: None. This blueprint runs out of the box — every external system is modeled inside the same service.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` to change the system name, model provider, or business specifics.
3. In Claude Code, from inside the folder, run:

   ```
   /akka:specify @SPEC.md
   ```

That single command scaffolds the specification and then auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` (see `SPEC.md` Section 12). When the service is running, Claude prints the listening URL.

## What you'll get

- Four AutonomousAgents: `IntakeAgent`, `ResearchAgent`, `ScoringAgent`, `StrategyAgent`.
- One `LeadStrategyWorkflow` orchestrating intake → research → score → strategy as a linear pipeline.
- Two EventSourcedEntities: `LeadEntity` (per-lead lifecycle) and `InboundLeadQueue`.
- One `LeadsView` read model with an SSE stream.
- Two Consumers: `InboundLeadConsumer` (starts a workflow per inbound lead) and `ScoreEvalConsumer` (runs an automatic eval on each scored lead).
- One TimedAction: `LeadSimulator` (drips sample leads).
- Two HttpEndpoints: `LeadEndpoint` (the `/api` surface) and `AppEndpoint` (the embedded UI).

## Customise before generating

- System name and short name — `SPEC.md` Section 1 and the browser title.
- Model provider — `SPEC.md` Section 11 lists the key-sourcing options.
- Domain specifics — the agent prompts under `prompts/`, the scoring rubric, and the sample leads.

## What gets validated

The acceptance journeys in `reference/user-journeys.md`:

- Submit a lead and watch it move through `INTAKE` → `RESEARCHED` → `SCORED` → `COMPLETE`.
- A scored lead triggers an automatic eval that records an accuracy/fairness result.
- A completed lead carries a non-empty engagement strategy.
- Sample leads dripped by the simulator progress through the pipeline with no manual submission.

## License

Apache 2.0.
