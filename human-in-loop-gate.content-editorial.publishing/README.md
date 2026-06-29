# Akka Sample: HITL Blog Publishing

A blog post is drafted by an agent, paused at a human approval gate, and published by a second agent only after a person approves it.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.content-editorial.publishing  ~/my-projects/publishing
cd ~/my-projects/publishing
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `ContentAgent` (AutonomousAgent) — drafts a blog post on a topic, returns a typed `DraftPost{title, content}`.
- `PublishingAgent` (AutonomousAgent) — publishes an approved post, returns a typed `PublishedPost{url, publishedAt}`.
- `PublishingWorkflow` (Workflow) — a 3-task graph: draft → await approval → publish.
- `PostEntity` (EventSourcedEntity) — the post lifecycle and its events.
- `PostsView` (View) — a read model the UI queries and streams over SSE.
- `PublishingEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/content-agent.md` — the drafting instructions and tone.

## What gets validated

- Submitting a topic drafts a post that appears in `DRAFTED` with non-empty content.
- Approving a `DRAFTED` post drives it to `PUBLISHED` with a non-null URL.
- Rejecting a `DRAFTED` post drives it to a terminal `REJECTED` state with the reason shown.
- The publish step never runs unless the post is `APPROVED`.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
