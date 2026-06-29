# Akka Sample: Content Creator Flow

One topic in; a research report, a blog post, and a LinkedIn post out — each brand-checked before it surfaces and each scored against brand guidelines.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One AI provider key, sourced at generation time (you will not write the value to disk): `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, the generator offers a mock provider that needs no key.
- Host software: none. This blueprint runs out of the box — every external surface is modeled inside the same service.

## Generate the system

```sh
cp -r ./composite-multi-team.content-editorial.content-creator-flow  ~/my-projects/content-creator-flow
cd ~/my-projects/content-creator-flow
```

(Optional) Edit `SPEC.md` — system name, model provider, the canned topics.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. Section 12 of `SPEC.md` tells Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` without stopping, and to print the listening URL when the service is up.

## What you'll get

- Three writer agents — a research analyst, a blog writer, a LinkedIn writer — each producing a typed artifact from one topic.
- A brand reviewer agent wired as a before-response guardrail that blocks off-brand content.
- A quality evaluator agent that scores each finished campaign against brand guidelines.
- A `ContentWorkflow` orchestrating research → drafting → review → evaluation.
- A `CampaignEntity` (event-sourced) holding the topic, the three outputs, and the review verdict.
- A `CampaignsView` (CQRS read model) the UI queries and streams over SSE.
- A topic simulator (TimedAction) and a queue Consumer that start a fresh campaign per topic.
- Two HTTP endpoints and a single self-contained UI with the five standard tabs.

## Customise before generating

- `SPEC.md` Section 1 — system name and one-line pitch.
- `SPEC.md` Section 11 — model provider and HTTP port.
- `prompts/*.md` — the voice and brand guidelines each agent writes against.
- `src/main/resources/sample-events/topics.jsonl` (created at generation) — the canned topics the simulator drips.

## What gets validated

- Submitting a topic produces a campaign that moves through research, drafting, review, and evaluation to `COMPLETED`, with all three outputs populated.
- A topic flagged off-brand is held at `BLOCKED` with a reason, and no outputs surface.
- The simulator seeds campaigns on its own with no UI interaction.
- The Eval Matrix and Risk Survey tabs render the governance posture; the App UI tab streams campaigns over SSE.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
