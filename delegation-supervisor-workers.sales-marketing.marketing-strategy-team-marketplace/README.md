# Akka Sample: Marketing Strategy Team

A campaign director delegates discrete strategy work to four specialist agents — market research, audience targeting, messaging, and channel planning — then merges their outputs into one cohesive campaign brief. Demonstrates the **delegation-supervisor-workers** coordination pattern with an embedded brand and legal compliance guardrail that vets marketing claims before they reach the user.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- Host software: none. This blueprint runs out of the box — the campaign brief pipeline and the marketing tools are modelled inside the same service.
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.sales-marketing.marketing-strategy-team-marketplace  ~/my-projects/marketing-strategy
cd ~/my-projects/marketing-strategy
```

(Optional) Edit `SPEC.md` to change the artifact name, model provider, or campaign output schema.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **CampaignDirector** — AutonomousAgent that decomposes a campaign objective into a strategy plan and assembles the final campaign brief.
- **MarketResearcher** — AutonomousAgent that gathers market insights and competitive context.
- **AudienceTargeter** — AutonomousAgent that defines audience segments and persona profiles.
- **MessageStrategist** — AutonomousAgent that drafts key messages and brand-aligned copy direction.
- **ChannelPlanner** — AutonomousAgent that selects and prioritises marketing channels with a rationale.
- **CampaignWorkflow** — Workflow that fans the work out to all four worker agents in parallel, then calls CampaignDirector for assembly.
- **CampaignBriefEntity** — EventSourcedEntity holding the full campaign brief lifecycle.
- **CampaignView** — projection the UI streams via SSE.
- **CampaignEndpoint + AppEndpoint** — REST + SSE + static UI serving.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — change the campaign objectives the simulator drips, or remove the simulator entirely.
- `SPEC.md §5` — adjust the `CampaignBrief` record fields (e.g., add `budgetEstimate`).
- `prompts/market-researcher.md` — narrow the agent to a specific industry vertical.
- `eval-matrix.yaml` — add a `before-tool-call` guardrail if you wire a real ad-platform API.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. Submit a campaign objective → brief enters `PLANNING`, then `IN_PROGRESS`, then `ASSEMBLED`.
2. Workers fail-fast → if any worker times out, the brief enters `DEGRADED` with whichever partial outputs exist.
3. The compliance guardrail blocks a brief containing non-compliant marketing claims; the brief enters `BLOCKED`.
4. Wait after a successful assembly; the brief's row in the UI shows an eval score.

## License

Apache 2.0.
