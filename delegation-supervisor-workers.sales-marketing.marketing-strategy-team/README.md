# Akka Sample: Marketing Strategy Team

Type a project brief. A lead strategist delegates research, strategy, and content work to specialist agents, then assembles a grounded marketing plan.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One model-provider key option (you pick during generation): an existing `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`; a path to an env file you maintain; a secrets-store URI; a key typed once into the session; or a mock provider that needs no key.
- Host software: none. This blueprint runs out of the box — the web-search grounding source is modeled inside the same service.

## Generate the system

```sh
cp -r ./delegation-supervisor-workers.sales-marketing.marketing-strategy-team  ~/my-projects/marketing-strategy-team
cd ~/my-projects/marketing-strategy-team
```

(Optional) Edit `SPEC.md` to change the system name, model provider, or the canned briefs.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- Four collaborating agents: a `CampaignLeadAgent` supervisor plus `MarketResearchAgent`, `StrategyAgent`, and `ContentAgent` workers.
- A `StrategyWorkflow` that delegates research → strategy → content and assembles the result.
- A `CampaignEntity` event-sourced lifecycle and a `CampaignsView` read model streamed to the UI over SSE.
- An in-process `WebSearchEndpoint` that grounds research in canned source results.
- A brief intake path: `BriefQueue` entity, `BriefConsumer`, and a `BriefSimulator` that drips canned briefs.
- A claim-grounding guardrail and a self-evaluation eval-event, surfaced on the Eval Matrix tab.
- A single self-contained UI with five tabs.

## Customise before generating

- System name and pitch — SPEC.md Section 1.
- Model provider — SPEC.md Section 11 ("Generation workflow").
- Canned project briefs and search results — the resource files named in SPEC.md Section 11 ("Companion files").

## What gets validated

- Submit a brief and watch a grounded campaign plan assemble (J1).
- The lead agent's plan cites only claims present in the research findings (J2).
- A self-evaluation score against the brief appears on the assembled plan (J3).
- The simulator seeds briefs on its own without any UI interaction (J4).

See `reference/user-journeys.md` for the full acceptance steps.

## License

Apache 2.0.
