# Akka Sample: Story Teller

A single story-generation agent takes a user prompt, an optional genre, and optional style constraints, and returns a fully-formed short story with a title, narrative body, and a brief author's note explaining the choices made.

Demonstrates the **single-agent** coordination pattern in the content-editorial domain. One `StoryTellerAgent` does all creative work; the surrounding components handle prompt shaping, output validation, and quality scoring — none of them touch the model directly.

## Prerequisites

- Claude Code installed.
- Akka plugin enabled (`/akka:setup`). See [doc.akka.io](https://doc.akka.io/) for "Spec-Driven Development with Claude Code".
- A model-provider key — **or** none, if you pick the mock LLM during scaffolding. If you do have a key, supply it via one of: an existing shell env var (`ANTHROPIC_API_KEY` / `OPENAI_API_KEY` / `GOOGLE_AI_GEMINI_API_KEY`), an env file you maintain, a secrets-store URI, or a one-time prompt during `/akka:specify`. Akka never writes the key value to disk.
- No additional host software. The blueprint runs out of the box — the story corpus lives in-process and the agent's tool calls are simulated.

## Generate the system

```sh
cp -r ./single-agent.content-editorial.story-teller  ~/my-projects/story-teller
cd ~/my-projects/story-teller
```

(Optional) Edit `SPEC.md` to point at a different genre set or to swap the default length constraints.

In Claude Code, from inside the blueprint folder:

```
/akka:specify @SPEC.md
```

That's the only command you type. SPEC.md's Section 12 instructs Claude to continue automatically through `/akka:plan` → `/akka:tasks` → `/akka:implement` → `/akka:build` after scaffolding completes. When `/akka:build` reports the service is up, Claude prints the listening URL and stops.

## What you'll get

- **StoryTellerAgent** — an AutonomousAgent that accepts a story prompt and style constraints and returns a typed `GeneratedStory`.
- **StoryWorkflow** — orchestrates prompt-enrichment → generation → quality-score per request.
- **StoryEntity** — an EventSourcedEntity holding the per-story lifecycle.
- **PromptEnricher** — a Consumer that subscribes to `StoryRequested` events, applies content-safety checks and metadata tagging, and emits `PromptEnriched` back to the entity.
- **StoryView + StoryEndpoint + AppEndpoint** — read model, REST/SSE API, and the embedded UI.
- A 5-tab embedded UI: Overview / Architecture / Risk Survey / Eval Matrix / App UI.

## Customise before generating

- `SPEC.md §3` — swap the seeded genre and prompt examples for your own (the JSONL file under `src/main/resources/sample-events/seed-prompts.jsonl` after generation).
- `SPEC.md §5` — extend `GeneratedStory` with additional fields (e.g., `targetAgeGroup`, `readingLevel`, `thematicTags`).
- `prompts/story-teller.md` — narrow the agent's role (an educational deployer would constrain output to age-appropriate vocabulary; a game-narrative deployer to a specific world-building canon).
- `eval-matrix.yaml` — wire a real content-safety classifier by naming it under the safety-filter mechanism's implementation paragraph.

## What gets validated

The user journeys in `reference/user-journeys.md` define the acceptance bar:

1. A user submits a prompt → the enricher runs → the agent generates a story → the story appears in the UI with a quality score.
2. The agent returns a malformed story on first try → the `before-agent-response` guardrail rejects it → the agent retries → a well-formed story lands.
3. Every recorded story has an on-decision quality score visible on the same UI card.
4. A prompt containing disallowed content triggers the safety filter; the entity transitions to `BLOCKED` and the reason is shown in the UI.

## License

Apache 2.0.
