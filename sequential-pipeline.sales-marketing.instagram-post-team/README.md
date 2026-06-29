# Akka Sample: Instagram Post Team

A pipeline of agents turns one creative brief into finished Instagram post content — an image-generation prompt, a caption, and a set of hashtags — running a brand and platform-policy check before the content is marked ready.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- An AI provider key: one of `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY`. If none is set, `/akka:specify` offers a mock-model option that needs no key.
- Host software: none. This sample runs out of the box.

## Generate the system

```sh
cp -r ./sequential-pipeline.sales-marketing.instagram-post-team ~/my-projects/instagram-post-team
cd ~/my-projects/instagram-post-team
```

(Optional) Edit `SPEC.md` — system name, model provider, brand-voice rules.

In Claude Code, from inside the folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` and print the listening URL when the service is up.

## What you'll get

- Two agents: a caption agent (caption + hashtags) and an image-prompt agent (image-generation prompt).
- A workflow chaining caption → image prompt → brand/safety check → ready.
- An event-sourced post entity, a read-model view, and an SSE stream to the UI.
- A brief simulator as a scheduled action.
- A before-agent-response guardrail that checks brand voice and platform policy before content is marked ready.
- A single-file UI with five tabs: Overview, Architecture, Risk Survey, Eval Matrix, App UI.

## Customise before generating

- `SPEC.md` Section 1 — system name and pitch.
- `SPEC.md` Section 11 — model provider and port.
- `prompts/caption-agent.md` and `prompts/image-prompt-agent.md` — brand voice, tone, hashtag policy.
- `eval-matrix.yaml` — the brand and platform-policy rules the before-agent-response guardrail enforces.

## What gets validated

- Submitting a brief produces a post that moves through composing and prompting to ready with a non-empty caption, hashtags, and image prompt.
- Content that breaks brand voice or platform policy is blocked before it is marked ready.
- The brief simulator seeds new briefs on its own and the pipeline runs each to completion.

See `reference/user-journeys.md`.

## License

Apache 2.0.
