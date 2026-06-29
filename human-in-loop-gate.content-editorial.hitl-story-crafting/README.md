# Akka Sample: HITL Story Crafting

An agent drafts one chapter of an interactive story per turn. The workflow pauses after each draft and waits for the reader to provide direction before the next chapter is written.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) → "Spec-Driven Development with Claude Code".
- One of: `ANTHROPIC_API_KEY`, `OPENAI_API_KEY`, or `GOOGLE_AI_GEMINI_API_KEY` — or pick the mock provider during generation (no key needed).
- Host software: None. This blueprint runs out of the box.

## Generate the system

```sh
cp -r ./human-in-loop-gate.content-editorial.hitl-story-crafting  ~/my-projects/hitl-story-crafting
cd ~/my-projects/hitl-story-crafting
```

(Optional) Edit `SPEC.md` to change the system name or model provider.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That is the only command. SPEC.md Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding, and to print the listening URL when the service is up.

## What you'll get

- `StoryDrafterAgent` (AutonomousAgent) — writes the next chapter given the story so far and reader direction; returns a typed `ChapterDraft{title, body}`.
- `ContentGuardAgent` (AutonomousAgent) — screens a chapter draft for content policy before it reaches the reader; returns a typed `GuardResult{passed, reason}`.
- `StoryWorkflow` (Workflow) — a repeating 3-task graph: draft chapter → screen draft → await reader direction.
- `StoryEntity` (EventSourcedEntity) — the story lifecycle and its events.
- `StoriesView` (View) — a read model the UI queries and streams over SSE.
- `StoryEndpoint` + `AppEndpoint` (HttpEndpoints) — the REST/SSE surface and the static UI.

## Customise before generating

- `SPEC.md` Section 1 — the system name.
- `SPEC.md` Section 11 — the model provider and HTTP port.
- `prompts/story-drafter-agent.md` — the narrative style and content constraints.

## What gets validated

- Starting a story with a premise creates a chapter in `AWAITING_DIRECTION` with non-empty body.
- Providing direction continues to the next chapter still in `AWAITING_DIRECTION`.
- Ending the story transitions it to terminal `COMPLETED`.
- A chapter that fails the content screen is never shown to the reader; the story enters `SCREENED_OUT`.
- The next chapter never starts until the reader provides direction.

Full journeys: `reference/user-journeys.md`.

## License

Apache 2.0.
