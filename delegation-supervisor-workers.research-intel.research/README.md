# Akka Sample: Multi-Agent Research Brief

A research coordinator delegates findings-gathering and analytical work to two specialist agents running **in parallel**, then merges their outputs into one unified research brief. Demonstrates the **delegation-supervisor-workers** coordination pattern with embedded governance.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound topic stream and the research tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.research-intel.research  ~/my-projects/research
cd ~/my-projects/research
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **ResearchCoordinator** — AutonomousAgent that decomposes a topic into a research plan and synthesises results.
- **Researcher** — AutonomousAgent that gathers findings (web-search-like behaviour, modelled with seeded tools).
- **Analyst** — AutonomousAgent that interprets the question and proposes implications.
- **ResearchWorkflow** — Workflow that fans the work out to Researcher and Analyst in parallel, then calls Coordinator for synthesis.
- **ResearchBriefEntity** — EventSourcedEntity holding the full brief lifecycle.
- **ResearchView** — projection the UI streams via SSE.
- **ResearchEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the brief topics the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `ResearchBrief` record fields (e.g., add `confidenceScore`).
- `prompts/researcher.md` — narrow the agent to a single domain.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real search API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a research topic → brief enters `PLANNING`, then `IN_PROGRESS`, then `SYNTHESISED`.
2. Workers fail-fast → if either Researcher or Analyst times out, the brief enters `DEGRADED` with whichever partial output exists.
3. Eval-event sampling captures one synthesis decision and surfaces the score on the App UI.

## License

Apache 2.0.
