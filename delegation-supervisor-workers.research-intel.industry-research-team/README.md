# Akka Sample: Industry Research Team

A supervisor agent splits a research request into industry sectors, delegates each sector to a worker agent, then synthesizes the findings into one grounded research brief.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- A model key, or none. You can run with a mock model and no key. To use a real model, have one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` available to the Claude Code session (the generator offers five key-sourcing options and never writes the value to disk).
- Host software: None. This blueprint runs out of the box — every external surface is modeled inside the same service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.industry-research-team  ~/my-projects/industry-research-team
cd ~/my-projects/industry-research-team
```

(Optional) Edit `SPEC.md` — system name, model provider, the sector list, the disclaimer text.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, then print the listening URL.

## What you'll get

- Two AutonomousAgents: a `SupervisorAgent` that plans and synthesizes, and a `SectorAnalystAgent` worker invoked once per assigned sector.
- A `ResearchWorkflow` orchestrating plan → analyze → synthesize → sanitize.
- A `BriefEntity` (event-sourced) holding each brief's lifecycle and a `BriefsView` CQRS read model.
- A `RequestConsumer` plus a `RequestSimulator` TimedAction that drips canned requests from a JSONL file.
- Two HttpEndpoints (the JSON API + the static UI) and a five-tab browser UI.
- A before-agent-response grounding guardrail and a sector-disclaimer sanitizer.

## Customise before generating

- `SPEC.md` Section 1 — system name and one-line pitch.
- `SPEC.md` Section 11 — model provider, HTTP port, the canned sector list.
- `prompts/supervisor-agent.md` and `prompts/sector-analyst-agent.md` — agent behavior and the grounding bar.
- `eval-matrix.yaml` — the disclaimer text the sanitizer appends.

## What gets validated

The user journeys in `reference/user-journeys.md`:

- Submit a research request and watch it produce a sector-grounded brief.
- An ungrounded synthesis is blocked before it reaches the deployer.
- The sanitizer appends the required sector disclaimer to a completed brief.
- The simulator drives briefs end to end with no UI interaction.

## License

Apache 2.0.
