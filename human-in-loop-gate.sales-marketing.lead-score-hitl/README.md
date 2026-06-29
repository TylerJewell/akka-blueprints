# Akka Sample: Lead Score HITL

A lead pipeline that collects, analyzes, and scores inbound leads, pauses for a human to approve the top-ranked few, then drafts outreach email for the approved leads.

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

- Four AutonomousAgents: `LeadCollectionAgent`, `LeadAnalysisAgent`, `LeadScoringAgent`, `OutreachAgent`.
- One `LeadScoringWorkflow` orchestrating collect → analyze → score → await-review → outreach.
- Two EventSourcedEntities: `LeadEntity` (per-lead lifecycle) and `InboundLeadQueue`.
- One `LeadsView` read model with an SSE stream.
- One `InboundLeadConsumer` that starts a workflow per inbound lead.
- Two TimedActions: `LeadSimulator` (drips sample leads) and `LeadRankingMonitor` (shortlists the top-ranked leads).
- Two HttpEndpoints: `LeadEndpoint` (the `/api` surface) and `AppEndpoint` (the embedded UI).

## Customise before generating

- System name and short name — `SPEC.md` Section 1 and the browser title.
- Model provider — `SPEC.md` Section 11 lists the key-sourcing options.
- Domain specifics — the agent prompts under `prompts/`, the scoring threshold, and the sample leads.

## What gets validated

The acceptance journeys in `reference/user-journeys.md`:

- Submit a lead and watch it move through `COLLECTED` → `ANALYZED` → `SCORED`.
- A top-ranked lead is shortlisted and surfaces for review.
- Approve a shortlisted lead and watch outreach email get drafted.
- Reject a shortlisted lead and watch it reach a terminal state with no outreach.

## License

Apache 2.0.
