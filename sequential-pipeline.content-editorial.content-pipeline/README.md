# Akka Sample: Content Pipeline

Submit a topic; a research agent gathers sources, a writer agent drafts an article, and a self-critique pass scores it before publishing.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation, which needs no key.
- Host software: None. This blueprint runs out of the box; the web-search source is modeled inside the same service over canned data.

## Generate the system

```sh
cp -r ./sequential-pipeline.content-editorial.content-pipeline  ~/my-projects/content-pipeline
cd ~/my-projects/content-pipeline
```

(Optional) Edit `SPEC.md` — system name, model provider, sample topics.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding.

## What you'll get

- Two AutonomousAgents: `ResearchAgent` (topic → research brief) and `WriterAgent` (brief → article draft).
- One Agent: `CritiqueAgent` (self-critique score on the draft).
- One Workflow: `ContentPipelineWorkflow` chaining research → write → critique → publish.
- One EventSourcedEntity: `ArticleEntity` holding the article lifecycle.
- One EventSourcedEntity: `InboundTopicQueue` for topic intake.
- One View: `ArticlesView` projecting article state for the UI.
- One Consumer: `TopicConsumer` starting a workflow per topic.
- One TimedAction: `TopicSimulator` dripping canned topics.
- Two HttpEndpoints: `ContentEndpoint` (API + metadata) and `AppEndpoint` (static UI).

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider, port, sample topics file.
- `prompts/` — the research, writer, and critique agent prompts.

## What gets validated

- Submitting a topic produces a published article with a non-empty body via the live pipeline.
- The before-tool-call guard blocks a research source outside the allowed-domain list.
- The before-agent-response guard blocks a draft that fails the style/safety bar.
- The critique pass records a score visible in the UI before publishing.
- The simulator seeds topics with no UI interaction.

See `reference/user-journeys.md`.

## License

Apache 2.0.
