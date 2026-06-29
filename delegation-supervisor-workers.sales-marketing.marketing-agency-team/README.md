# Akka Sample: Marketing Agency Team

A campaign director delegates a website launch and a marketing strategy brief to two specialist agents running **in parallel**, then merges their outputs into a coordinated marketing plan. Demonstrates the **delegation-supervisor-workers** coordination pattern with an embedded brand-safety guardrail.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the inbound campaign request stream and the marketing tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.sales-marketing.marketing-agency-team  ~/my-projects/marketing-agency-team
cd ~/my-projects/marketing-agency-team
```

(Optional) Edit `SPEC.md` to change the campaign scenarios the simulator drips, or remove the simulator entirely.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CampaignDirector** — AutonomousAgent that decomposes a campaign brief and synthesises the final marketing plan.
- **WebsiteLauncher** — AutonomousAgent that produces a website launch checklist and launch brief.
- **StrategyAdvisor** — AutonomousAgent that produces a positioning strategy and messaging framework.
- **CampaignWorkflow** — Workflow that fans the work out to WebsiteLauncher and StrategyAdvisor in parallel, then calls CampaignDirector for synthesis.
- **CampaignPlanEntity** — EventSourcedEntity holding the full plan lifecycle.
- **CampaignView** — projection the UI streams via SSE.
- **CampaignEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the campaign scenarios the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `CampaignPlan` record fields (e.g., add `budgetEstimate`).
- `prompts/website-launcher.md` — narrow the agent to a specific CMS or channel.
- `eval-matrix.yaml` — add an `on-decision-eval` control if you wire a real channel-performance API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a campaign brief → plan enters `PLANNING`, then `IN_PROGRESS`, then `SYNTHESISED`.
2. Workers fail-fast → if either WebsiteLauncher or StrategyAdvisor times out, the plan enters `DEGRADED` with whichever partial output exists.
3. Brand guardrail fires → if the synthesised plan contains off-brand or non-compliant copy, the plan enters `BLOCKED`.

## License

Apache 2.0.
