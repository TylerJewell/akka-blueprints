# Akka Sample: Consulting Coordinator

A coordinator routes each client engagement: routine research goes to a junior researcher; high-stakes work is handed off to a senior consultant whose recommendation gets a compliance review.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider when `/akka:specify` asks, and no key is needed.
- Host software: None. This blueprint runs out of the box; the external client intake and compliance desk are modeled inside the same Akka service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.consulting  ~/my-projects/consulting-coordinator
cd ~/my-projects/consulting-coordinator
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the engagement domain.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- A `ConsultingCoordinator` agent that classifies each engagement and returns a routing decision.
- A `JuniorResearcher` autonomous agent that produces a research brief for routine work.
- A `SeniorConsultant` autonomous agent that produces a client recommendation for high-stakes work.
- An `EngagementWorkflow` that routes, runs the chosen worker, and records delivery.
- An `EngagementEntity` (event-sourced) plus an `EngagementsView` read model streamed to the UI.
- A routing-decision eval, an output guardrail on client-facing text, and a non-blocking compliance review of senior recommendations.
- A self-contained 5-tab UI served from the same service.

## Customise before generating

- **System name / model provider** — SPEC.md Section 1 and Section 11 (identity block).
- **Engagement domain** — the prompts under `prompts/` and the sample intake lines drive what kinds of engagements arrive; edit them for your sector.
- **Routing threshold** — the coordinator prompt defines what counts as high-stakes; adjust the criteria there.

## What gets validated

The journeys in `reference/user-journeys.md`:

- A routine engagement is delegated to the junior researcher and delivered.
- A high-stakes engagement is handed off to the senior consultant and delivered.
- A senior recommendation receives a non-blocking compliance review.
- The routing decision carries an eval score in the UI.

## License

Apache 2.0.
