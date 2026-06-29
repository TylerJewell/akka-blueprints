# Akka Sample: Landing Page Generator

Type a one-line product concept. Three agents run in sequence and hand back a finished Tailwind landing page.

## Prerequisites

- Claude Code with the Akka plugin installed. Install docs: <https://doc.akka.io/>.
- One model-provider API key, sourced at generation time via one of five options (mock, env-var name, env-file path, secrets URI, or type-once) — see `SPEC.md` Section 11. You never write the key value to disk.
- Host software: None. This blueprint runs out of the box — search and scraping are modeled in-process with Akka components over canned data.

## Generate the system

1. Copy this folder into your own project location.
2. Optionally edit `SPEC.md` — system name, model provider, agent prompts.
3. In Claude Code, run:

```
/akka:specify @SPEC.md
```

`SPEC.md` Section 12 auto-chains `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build`. When the service is up, Claude prints the listening URL.

## What you'll get

- Three agents: `IdeaAnalyst`, `TemplateSelector`, `CustomizationAgent`.
- `GenerationWorkflow` coordinating analyze → select → customize → validate.
- `PageEntity` (event-sourced page lifecycle) and `InboundRequestQueue`.
- `PagesView` read model with SSE streaming to the UI.
- `RequestConsumer`, `RequestSimulator` (drips canned concepts).
- `GenerationEndpoint`, `ScraperEndpoint` (in-process template source), `AppEndpoint`.
- A single self-contained 5-tab UI at `src/main/resources/static-resources/index.html`.

## Customise before generating

- System name and pitch — `SPEC.md` Section 1.
- Model provider default — `SPEC.md` Section 11 and the generated `application.conf`.
- Agent behavior — `prompts/idea-analyst.md`, `prompts/template-selector.md`, `prompts/customization-agent.md`.

## What gets validated

The journeys in `reference/user-journeys.md`:

- Submit a concept and watch a page reach `PUBLISHED`.
- A blocked off-allowlist scrape keeps the run on canned sources.
- Unsafe generated markup is sanitized before it is stored.
- Markup that fails the lint gate transitions to `REJECTED`.

## License

Apache 2.0.
